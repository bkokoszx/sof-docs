.. _parameter-propagation:

Parameter Propagation
#####################

Sound Open Firmware defines pipelines that can accept a range of runtime
parameters (e.g. rate, format, channels). These parameters and any
modifications that can be made by components are passed to components 
buffers in following steps.

#. Find DAI component in pipeline and fetch its hardware stream parameters set during ``dai_set_config()``.

#. Walk from DAI to all other endpoints in order to pass its hardware stream parameters fetched in previous step.

#. Walk from HOST to all other endpoints in order to refine buffers parameters (set in step 2)
   with PCM parameters that were sent by driver in ``ipc_stream_pcm_params()``. In these step
   every component decides whether it can convert PCM parameters to DAI hardware
   parameters.

//https://github.com/thesofproject/sof/pull/2228#discussion_r366767891

Step 1. and 2. are implemented by non-tail recursive ``pipeline_comp_hw_params()``
function, which uses ``pipeline_for_each_comp()`` like any other ``pipeline_comp_*()``
functions. It always starts from host and goes downstream for playback and upstream
for capture. At the beginning, ``pipeline_comp_hw_params()`` recursively
goes from HOST to endpoints in order to find DAI component. When it finds
DAI it uses ``comp_dai_get_hw_params()`` to fetch its hardware stream
parameters.

.. code-block:: c

    /* Fetch hardware stream parameters from DAI component */
	if (current->comp.type == SOF_COMP_DAI) {
		ret = comp_dai_get_hw_params(current,
					     &ppl_data->params->params);
		if (ret < 0) {
			trace_pipe_error("pipeline_find_dai_comp(): comp_dai_get_hw_params() error.");
			return ret;
		}
	}

Step 3. is implemented by ``pipeline_comp_params()`` function. It takes
PCM parameters sent by driver and sends them to each component in pipeline -
from host downstream for playback and upstream for capture. Each component
tries to refine parameters already set in previous step with new ones in
``verify_params()`` function (e.g. ``src_verify_params()``). Some components
are able to support different input and output format (e.g. ``SRC``
component is able to convert stream rate), so they are responsible
for corretly setting sink/source parameters.
 

Fetching and setting hardware stream parameters
***********************************************

``comp_dai_get_hw_params()`` function invokes ``dai_get_hw_params()``
from components ops. It is implemented only for DAI component. For any other 
component it returns an error.

.. code-block:: c

   static inline int comp_dai_get_hw_params(struct comp_dev *dev, struct sof_ipc_stream_params *params)
   {
		if (dev->drv->ops.dai_get_hw_params)
			return dev->drv->ops.dai_get_hw_params(dev, params);
		return -EINVAL;
   }

The implementation of ``dai_get_hw_params()`` for DAI component is
``dai_comp_get_hw_params()`` function, which in turn invokes ``get_hw_params()``
function (placed in dai_driver) specific for each physical DAI (e.g. SSP, ALH, HDA, DMIC). 
For example, for SSP DAI we have got following ``get_hw_params()`` function.

.. code-block:: c

	static int ssp_get_hw_params(struct dai *dai, struct sof_ipc_stream_params  *params)
	{
		struct ssp_pdata *ssp = dai_get_drvdata(dai);

		params->rate = ssp->params.fsync_rate;
		params->channels = ssp->params.tdm_slots;
		params->buffer_fmt = 0;

		switch (ssp->params.sample_valid_bits) {
		case 16:
			params->frame_fmt = SOF_IPC_FRAME_S16_LE;
			break;
		case 24:
			params->frame_fmt = SOF_IPC_FRAME_S24_4LE;
			break;
		case 32:
			params->frame_fmt = SOF_IPC_FRAME_S32_LE;
			break;
		default:
			trace_ssp_error("ssp_get_hw_params(): not supported format");
			return -EINVAL;
		}

		return 0;
	}

Above SSP function updates ``params`` variable with its private hardware parameters
(``ssp->params``) set during ``dai_set_config()``. There are several DAI interfaces
(e.g. ALH, HDA) that do nothing during ``dai_set_config()``. There is no information
about how they were configured, so we can assume that we can play anything (only need is
to configure dma in the same way on the firmware and host side). In that case, we set
specific parameters to 0, which denotes that they can vary (below is presented HDA example):

.. code-block:: c

	static int hda_get_hw_params(struct dai *dai, struct sof_ipc_stream_params  *params)
	{
		/* 0 means variable */
		params->rate = 0;
		params->channels = 0;
		params->buffer_fmt = 0;
		params->frame_fmt = 0;

		return 0;
	}

After we fetch parameters from DAI, we set them in each buffer we passed
during searching DAI component (in ``pipeline_comp_hw_params()`` function).

.. code-block:: c

	/* set buffer parameters */
	if (calling_buf)
		buffer_set_params(calling_buf, &ppl_data->params->params, BUFFER_UPDATE_IF_UNSET);

Setting PCM parameters
**********************

Setting PCM parameters takes place in ``pipeline_comp_params()`` function.
As mentioned, it takes PCM parameters sent by driver and sends them to each
component in pipeline (using ``comp_params()`` function). It always goes
downstream for playback and upstream for capture. Each component tries to
refine sink/source parameters set earlier with hardware parameters
fetched from DAI interface.
As an example, we can see how this is done in SRC component, as it has capability
to change stream rate. In ``src_params()`` we invoke ``src_verify_params()``
function:

.. code-block:: c

	static int src_verify_params(struct comp_dev *dev, struct sof_ipc_stream_params *params)
	{
		struct sof_ipc_comp_src *src = COMP_GET_IPC(dev, sof_ipc_comp_src);
		int ret;

		tracev_src_with_ids(dev, "src_verify_params()");

		/* check whether params->rate (received from driver) are equal
		 * to src->source_rate (PLAYBACK) or src->sink_rate (CAPTURE) set during
		 * creating src component in src_new().
		 * src->source/sink_rate = 0 means that source/sink rate can vary.
		 */
		if (dev->direction == SOF_IPC_STREAM_PLAYBACK) {
			if ((params->rate != src->source_rate) && src->source_rate) {
				trace_src_error_with_ids(dev,
										 "src_verify_params(): error: runtime stream pcm rate does not match rate fetched from ipc."
										);
				return -EINVAL;
			}
		} else {
			if ((params->rate != src->sink_rate) && src->sink_rate) {
				trace_src_error_with_ids(dev,
										 "src_verify_params(): error: runtime stream pcm rate does not match rate fetched from ipc."
										 );
				return -EINVAL;
			}
		}

		/* update downstream (playback) or upstream (capture) buffer parameters
		 */
		ret = comp_verify_params(dev, BUFF_PARAMS_RATE, params);
		if (ret < 0) {
			trace_src_error_with_ids(dev,
									 "src_verify_params() error: comp_verify_params() failed"
									);
			return ret;
		}

		return 0;
	}

It's worth pointing out, that during creating SRC component there is possibility to
force sink/source rates. In ``sof_ipc_comp_src`` struct there are ``sink_rate`` and
``source_rate`` fields, which can be set in order to force those rates. When they are
equal to 0 it means that they can vary, so we will take PCM or hardware parameters.
In above ``src_verify_params()`` function, firstly we check whether PCM rate
parameter received from driver are equal to sink_rate (playback)/source_rate (capture)
(assuming that sink/source_rate in ``sof_ipc_comp_src`` is not equal 0 - means, as mentioned,
that it cannot vary). If not, firmware will return an error.
Subsequently, we invoke following generic ``comp_verify_params(dev, BUFF_PARAMS_RATE, params)``
function:

.. code-block:: c

	int comp_verify_params(struct comp_dev *dev, uint32_t flag, struct sof_ipc_stream_params *params)
	{
		struct list_item *buffer_list;
		struct list_item *source_list;
		struct list_item *sink_list;
		struct list_item *clist;
		struct comp_buffer *sinkb;
		struct comp_buffer *buf;
		int dir = dev->direction;

		if (!params) {
			trace_comp_error("comp_verify_params() error: !params");
			return -EINVAL;
		}

		source_list = comp_buffer_list(dev, PPL_DIR_UPSTREAM);
		sink_list = comp_buffer_list(dev, PPL_DIR_DOWNSTREAM);

		/* searching for endpoint component e.g. HOST, DETECT_TEST, which
		 * has only one sink or one source buffer.
		 */
		if (list_is_empty(source_list) != list_is_empty(sink_list)) {
			if (!list_is_empty(source_list))
				buf = list_first_item(&dev->bsource_list,
							  struct comp_buffer,
							  sink_list);
			else
				buf = list_first_item(&dev->bsink_list,
							  struct comp_buffer,
							  source_list);

			/* update specific pcm parameter with buffer parameter if
			 * specific flag is set.
			 */
			comp_update_params(flag, params, buf);

			/* overwrite buffer parameters with modified pcm
			 * parameters
			 */
			buffer_set_params(buf, params, BUFFER_UPDATE_FORCE);

			/* set component period frames */
			component_set_period_frames(dev, buf->stream.rate);
		} else {
			/* for other components we iterate over all downstream buffers
			 * (for playback) or upstream buffers (for capture).
			 */
			buffer_list = comp_buffer_list(dev, dir);

			list_for_item(clist, buffer_list) {
				buf = buffer_from_list(clist, struct comp_buffer,
							   dir);

				comp_update_params(flag, params, buf);

				buffer_set_params(buf, params, BUFFER_UPDATE_FORCE);
			}

			/* fetch sink buffer in order to calculate period frames */
			sinkb = list_first_item(&dev->bsink_list, struct comp_buffer,
						source_list);

			component_set_period_frames(dev, sinkb->stream.rate);
		}

		return 0;
	}


Function ``comp_verify_params()`` updates sink (playback) and source (capture)
parameters with PCM parameters given as one of arguments. We can also
specify whether we want to overwrite all of parameters. SRC component
is able to change stream rate, so we invoke ``comp_verify_params(dev, BUFF_PARAMS_RATE, params)``
with ``BUFF_PARAMS_RATE`` flag. It means that we don't want to change
sink (playback)/source (capture) rate earlier set in ``pipeline_comp_hw_params()``,
just beacause SRC can do rate conversion. Precisely, parameters overwritting
takes place in generic ``comp_update_params()`` function. 

