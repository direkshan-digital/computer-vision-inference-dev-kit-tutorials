# Tutorial Step 4: Using Asynchronous API

![image alt text](../doc_support/step4_image_0.png)

# Table of Contents

<p></p><div class="table-of-contents"><ul><li><a href="#tutorial-step-4-using-asynchronous-api">Tutorial Step 4: Using Asynchronous API</a></li><li><a href="#table-of-contents">Table of Contents</a></li><li><a href="#introduction">Introduction</a></li><li><a href="#using-the-asynchronous-api">Using the Asynchronous API</a><ul><li><a href="#command-line-arguments">Command Line Arguments</a></li><li><a href="#basedetection-class">BaseDetection Class</a><ul><li><a href="#basedetection">BaseDetection()</a></li><li><a href="#submitrequest">submitRequest()</a></li><li><a href="#wait">wait()</a></li><li><a href="#resultisready">resultIsReady()</a></li><li><a href="#requestsinprocess">requestsInProcess()</a></li><li><a href="#cansubmitrequest">canSubmitRequest()</a></li></ul></li><li><a href="#vehicledetection-class">VehicleDetection Class</a><ul><li><a href="#submitrequest">submitRequest()</a></li><li><a href="#enqueue">enqueue()</a></li><li><a href="#fetchresults">fetchResults()</a></li></ul></li><li><a href="#vehicleattribsdetection-class">VehicleAttribsDetection Class</a><ul><li><a href="#enqueue">enqueue()</a></li><li><a href="#fetchresults">fetchResults()</a></li></ul></li><li><a href="#main">main()</a><ul><li><a href="#report-async-mode">Report Async Mode</a></li><li><a href="#increase-storage">Increase Storage</a></li><li><a href="#add-pipeline-data-fields-and-storage">Add Pipeline Data Fields and Storage</a></li><li><a href="#main-loop">Main Loop</a><ul><li><a href="#pipeline-stage-0-prepare-and-start-inferring-a-batch-of-frames">Pipeline Stage #0: Prepare and Start Inferring a Batch of Frames</a></li><li><a href="#pipeline-stage-1-process-vehicles-inference-results">Pipeline Stage #1: Process Vehicles Inference Results</a></li><li><a href="#pipeline-stage-2-start-inferring-vehicle-attributes">Pipeline Stage #2: Start Inferring Vehicle Attributes</a></li><li><a href="#pipeline-stage-3-process-vehicle-attribute">Pipeline Stage #3: Process Vehicle Attribute</a></li><li><a href="#pipeline-stage-4-render-results">Pipeline Stage #4: Render Results</a></li><li><a href="#end-of-loop">End of Loop</a></li></ul></li></ul></li></ul></li><li><a href="#building-and-running">Building and Running</a><ul><li><a href="#build">Build</a></li><li><a href="#run">Run</a></li></ul></li><li><a href="#conclusion">Conclusion</a></li><li><a href="#navigation">Navigation</a></li></ul></div><p></p>

# Introduction

In Car Detection Tutorial Step 4, we will see how the code from Tutorial Step 3 has been modified to make use of the Inference Engine asynchronous API to make multiple inference requests without waiting for results.  The changes necessary include:

1. Add data structures to support multiple requests:

    1. Multiple request objects

    2. Multiple input data storage objects for each request

    3. Tracking objects for outstanding requests, data, etc.

2. Breakup previous stages #0 and #1 that ran inference synchronously into two new stages: "start inference" and “process results”

3. Change pipeline stages from running sequentially to making each stage data driven and run independently in the main loop:

    4. Stage #0 now first checks to see if new vehicle inference requests can be made, if so it prepares input image(s) and starts inference.

    5. Stage #1 now first checks to see if vehicle inference results are available and processes them, if not the main loop continues to the next stage.

    6. Stage #2 now first checks to see if there are vehicles available to infer as well as if vehicle attributes inference requests can be made, if so it prepares input vehicle(s) and starts inference.

    7. Stage #3 now first checks to see if vehicle attribute inference results are available and processes them, if not the main loop continues to the next stage.

    8. Stage #4 now first checks to see if all results are available and renders them, if not it continues the loop starting back with Stage #1.

# Using the Asynchronous API

In the Key Concepts section we learned the difference between the synchronous and asynchronous API.  Here we will see the changes necessary for asynchronous applied.  Below are code walkthroughs of the changes made to code from Tutorial Step 3 focusing primarily on the changes made rather than the entire code when possible.

1. Open up an Xterm window or use an existing window to get to a command shell prompt.

2. Change to the directory containing Tutorial Step 4:

```bash
cd tutorials/car_detection_tutorial/step_4
```


3. Open the files "main.cpp" and “car_detection.hpp” in the editor of your choice such as ‘gedit’, ‘gvim’, or ‘vim’.

## Command Line Arguments

The command line argument -n_async has been added to control how many outstanding API requests are to be allowed when running each model.  Setting -n_async to 1 is effectively the same as synchronous mode since later stages in the pipeline will not have anything to do until the results appear.  The following code in car_detection.hpp adds the -n_async argument to appear in the code as the variable FLAGS_n_async:

``` Cpp
/// @brief message async function flag
static const char async_depth_message[] = "Maximum number of outstanding async API calls allowed (1=synchronous=default, >1=asynchronous).";

/// \brief parameter to set depth (number of outstanding requests) of asynchronous API calls <br>
/// It is an optional parameter
DEFINE_uint32(n_async, 1, async_depth_message);
```


Later in main.cpp in a check is added to ParseAndCheckCommandLine() to ensure n_async is 1 or greater.

```cpp
    if (FLAGS_n_async < 1) {
        throw std::logic_error("Parameter -n_async must be >= 1");
    }
```


## BaseDetection Class

The BaseDetection class must be modified to allow for more than one request to exist.  First the single request storage is replaced with the following:

```cpp
    std::queue<InferRequest::Ptr> submittedRequests;
    std::vector<InferRequest::Ptr> requests;
    int inputRequestIdx;
    InferRequest::Ptr outputRequest;
    int maxSubmittedRequests;
```


In the code:

* submittedRequests - is a FIFO to track outstanding inference requests 

* requests - holds all the available inference requests that are reused

* inputRequestIdx - an index into ‘requests’ used to select the next one to use

* outputRequest - holds the current output request for retrieving outputs

* maxSubmittedRequests - the maximum number of outstanding requests allowed (set using -n_async)

### BaseDetection()

The constructor is changed to initialize the new storage variables:

```cpp
    BaseDetection(std::string &commandLineFlag, std::string topoName, int maxBatch)
        : commandLineFlag(commandLineFlag), topoName(topoName), maxBatch(maxBatch), maxSubmittedRequests(FLAGS_n_async),
		  plugin(nullptr), inputRequestIdx(0), outputRequest(nullptr), requests(FLAGS_n_async) {}
```


Note specifically ‘requests’ and maxSubmittedRequests being set to FLAGS_N_async, inputEequestIdx set to start from 0, and outputRequest set to nullptr (empty).

### submitRequest()

When submitting an asynchronous request:

1. The detector must be enabled and the current input request must not be null:

```cpp
virtual void submitRequest() {
        if (!enabled() || nullptr == requests[inputEequestIdx]) return;
```


2. Start the request asynchronously:

```cpp
       requests[inputEequestIdx]->StartAsync();
```


3. Record the request as outstanding:

```cpp
        submittedRequests.push(requests[inputEequestIdx]);
```


4. Cycle to the next input request to be used:

```cpp
        inputEequestIdx++;
        if (inputEequestIdx >= maxSubmittedRequests) {
        	inputEequestIdx = 0;
        }
    }
```


### wait()

When waiting for results of first (oldest) outstanding request:

1. Make sure detector is enabled:

```cpp
virtual void wait() {
        if (!enabled()) return;
```


2. Get the first outstanding request to wait on:

```cpp
        // get next request to wait on
        if (nullptr == outputRequest) {
```


3. Make sure there is an outstanding request:

```cpp
        	if (submittedRequests.size() < 1) return;
```


4. Get request and remove from tracking FIFO :

```cpp
        	outputRequest = submittedRequests.front();
        	submittedRequests.pop();
        }
```


5. Wait on the request until results are ready.  This is effectively a blocking synchronous call to Wait(), however other functions described below are used to check status without blocking.  

```cpp
        outputRequest->Wait(IInferRequest::WaitMode::RESULT_READY);
    }
```


### resultIsReady()

resultIsReady() has been added to tell when results are ready without blocking:

```cpp
    // call before wait() to check status
    bool resultIsReady() {
```


To check status, at least one outstanding request must be present:

```cpp
    	if (submittedRequests.size() < 1) return false;
```


Make non-blocking call to request’s Wait() to get status:

```Cpp
    	StatusCode state = submittedRequests.front()->Wait(IInferRequest::WaitMode::STATUS_ONLY);
		return (StatusCode::OK == state);
    }
```


### requestsInProcess()

requestsInProcess() has been added to tell when a request is being processed.

```Cpp
    bool requestsInProcess() {
```


Check the submitted requests FIFO, if it has a request in it then at least one requests is in process:

```cpp    
	// request is in progress if number of outstanding requests is > 0
    	return (submittedRequests.size() > 0);
    }
```


### canSubmitRequest()

canSubmitRequest() has been added to tell when a request can be made.

```cpp
    bool canSubmitRequest() {
```


If the number of outstanding requests is less than maximum, then a request can be made:

```Cpp
    	// ready when another request can be submitted
    	return (submittedRequests.size() < maxSubmittedRequests);
    }
```


## VehicleDetection Class

The VehicleDetection class needs to be updated to work with multiple requests.

### submitRequest()

submitRequest() no longer sets "resultsFetched" or clears “results”.

### enqueue()

enqueue() has the changes:

1. Instead of creating a single request, it will populate "requests[]" whenever a request object has not been created yet.

```cpp
        if (nullptr == requests[inputRequestIdx]) {
        	requests[inputRequestIdx] = net.CreateInferRequestPtr();
        }
```


2. Replace single "request" to use current input request “requests[inputRequestIdx]”:

```cpp
        auto  inputBlob = requests[inputRequestIdx]->GetBlob(input);
```


### fetchResults()

fetchResults() has the changes:

1. Replace "resultsFetched" with check that “outputRequest” is valid

```cpp
        if (nullptr == outputRequest) {
        	return;
        }
```


2. Replace single "request" to use “outputRequest” now set to current output request:

```cpp
        const float *detections = outputRequest->GetBlob(output)->buffer().as<float *>();
```


```cpp
\\ ...same processing of results...
```


3. Mark output request as done by setting to nullptr:

```Cpp
		// done with request
		outputRequest = nullptr;
```


## VehicleAttribsDetection Class

The VehicleAttribsDetection class needs to be updated to work with multiple requests.

### enqueue()

enqueue() has the changes:

3. Instead of creating a single request, it will populate "requests[]" whenever a request object has not been created yet.

```cpp
        if (nullptr == requests[inputRequestIdx]) {
        	requests[inputRequestIdx] = net.CreateInferRequestPtr();
        }
```


4. Replace single "request" to use current input request “requests[inputRequestIdx]”:

```cpp
        auto  inputBlob = requests[inputRequestIdx]->GetBlob(input);
```


### fetchResults()

fetchResults() has the changes:

4. Replace "resultsFetched" with check that “outputRequest” is valid

```cpp
        if (nullptr == outputRequest) {
        	return;
        }
```


5. Replace single "request" to use “outputRequest” now set to current output request:

```Cpp
			// 7 possible colors for each vehicle and we should select the one with the maximum probability
			const auto colorsValues = outputRequest->GetBlob(outputNameForColor)->buffer().as<float*>() + (bi * 7);
			// 4 possible types for each vehicle and we should select the one with the maximum probability
			const auto typesValues  = outputRequest->GetBlob(outputNameForType)->buffer().as<float*>() + (bi * 4);
```


```cpp
\\ ...same processing of results...
```


6. Mark output request as done by setting to nullptr:

```Cpp
		// done with request
		outputRequest = nullptr;
```


## main()

Before the main loop in main() there are changes to output asynchronous mode, storage, and data fields.

### Report Async Mode

Report value used for n_async and whether running asynchronously or synchronously.

```Cpp 
        const bool runningAsync = (FLAGS_n_async > 1);
        slog::info << "FLAGS_n_async=" << FLAGS_n_async << ", inference pipeline will operate "
        		<< (runningAsync ? "asynchronously" : "synchronously")
        		<< slog::endl;
```


### Increase Storage

More input frames must be stored with outstanding requests being made.  Increase total number to cover the maximum outstanding requests of batch size used:

```Cpp
        // read input (video) frames, need to keep multiple frames stored
        //  for batching and for when using asynchronous API.
        const int maxNumInputFrames = FLAGS_n_async * VehicleDetection.maxBatch + 1;  // +1 to avoid overwrite
```


### Add Pipeline Data Fields and Storage

1. Add "vehicleDetectionDone" and “vehicleAttributesDetectionDone” fields to indicate when inference is completed for both models.  Add “numVehiclesInferred” to track the number of inferred vehicles since Stage #2 and #3 are not complete until all vehicles have been run through vehicle attributes inference.

```cpp
		// structure to hold frame and associated data which are passed along
		//  from stage to stage for each to do its work
		typedef struct {
			std::vector<cv::Mat*> batchOfInputFrames;
			bool vehicleDetectionDone;
			cv::Mat* outputFrame;
			std::vector<cv::Rect> vehicleLocations;
			int numVehiclesInferred;
			std::vector<cv::Rect> licensePlateLocations;
			bool vehicleAttributesDetectionDone;
			std::vector<VehicleAttribsDetection::Attributes> vehicleAttributes;
		} FramePipelineFifoItem;
```


2. The previous stages that ran inference synchronously have been broken up into two new stages: "start inference" and “process results”.  Add more FIFOs to account for more stages:

```Cpp
		FramePipelineFifo pipeS2toS3Fifo;
		FramePipelineFifo pipeS3toS4Fifo;
```


3. Stage #3 will need to store vehicle attribute results in order to accumulate all the results coming from inference:

```cpp
		FramePipelineFifoItem accumVehAttribs;
		bool accumVehAttribsIsEmpty = true;
```


### Main Loop

The main loop sees changes to add new pipeline stages and to make each stage to be data driven and only run when there is something to do.

#### Pipeline Stage #0: Prepare and Start Inferring a Batch of Frames

Stage #0 from Tutorial Step 3 does both submit and wait for a inference request.  For asynchronous operation, it is broken up into Stage #0 to do the submit and Stage #1 to do the wait.  The new Stage #0 looks very much like the first half up through submitting the inference request with a couple changes:

1. In addition to if there are input frames still available ("haveMoreFrames"), Stage #0 now also checks to see if an input frame buffer is available via !inputFramePtrs.empty()  and that there is a request available to use via VehicleDetection.canSubmitRequest():

```Cpp
			/* *** Pipeline Stage 0: Prepare and Start Inferring a Batch of Frames *** */
        	// if there are more frames to do and a request available, then prepare and start batch
			if (haveMoreFrames && !inputFramePtrs.empty() && VehicleDetection.canSubmitRequest()) {
				// prepare a batch of frames
                                               // ... preparation code ...
```


2. Stage #0 now ends after submitting the inference request and passing the necessary data to Stag #1 vis pipeS0toS1Fifo.push(ps0s1i).

```Cpp
				if (numFrames > 0) {
					// start request
					t0 = std::chrono::high_resolution_clock::now();
					// start inference
					VehicleDetection.submitRequest();

					// queue data for next pipeline stage
					pipeS0toS1Fifo.push(ps0s1i);
				}
```


#### Pipeline Stage #1: Process Vehicles Inference Results

Stage #1 is responsible for checking for and then processing vehicle inference results started by Stage #0.  

1. First is to check whether there is work to be done.  When running synchronously (!runningAsync) if there is a request in progress, then enter the stage to wait for results.  When running asynchronously, check to see if a result is ready, then enter the stage and wait for the result (which will be a short wait).

```Cpp
			/* *** Pipeline Stage 1: Process Vehicles Inference Results *** */
			// sync: wait for results if a request was just submitted
			// async: if results are ready, then fetch and process in next stage of pipeline
			if ((!runningAsync && VehicleDetection.requestsInProcess()) || VehicleDetection.resultIsReady()) {
```


2. Wait for results:

```cpp
        		// wait for results, async will be ready
				VehicleDetection.wait();
				t1 = std::chrono::high_resolution_clock::now();
				detection_time = std::chrono::duration_cast<ms>(t1 - t0);
```


3. Get the results from the request storing in VehicleDetection.results:

```cpp
				// parse inference results internally (e.g. apply a threshold, etc)
				VehicleDetection.fetchResults();
```


4. For every request there is an input frame coming from Stage #0, get it form the FIFO:

```cpp
				// get associated data from last pipeline stage to use with results
				FramePipelineFifoItem ps0s1i = pipeS0toS1Fifo.front();
				pipeS0toS1Fifo.pop();
```


5. Each input frame in the batch that was input to the vehicle detection model will now become its own input frame going forward down the pipeline:

```				// prepare a FramePipelineFifoItem for each batched frame to get its detection results
				std::vector<FramePipelineFifoItem> batchedFifoItems;
				for (auto && bFrame : ps0s1i.batchOfInputFrames) {
					FramePipelineFifoItem fpfi;
					fpfi.outputFrame = bFrame;
					batchedFifoItems.push_back(fpfi);
				}
```


6. Go through the results putting detected vehicles and license plates with the associated input frame:

```Cpp
				// store results for next pipeline stage
				for (auto && result : VehicleDetection.results) {
					FramePipelineFifoItem& fpfi = batchedFifoItems[result.batchIndex];
					if (result.label == 1) {  // vehicle
						fpfi.vehicleLocations.push_back(result.location);
					} else { // license plate
						fpfi.licensePlateLocations.push_back(result.location);
					}
				}
```


7. Clear out the results for later re-use:

```Cpp
				// done with results, clear them
				VehicleDetection.results.clear();
```


8. For each of the input frames, send it to Stage #2:

```cpp
				// queue up output for next pipeline stage to process
				for (auto && item : batchedFifoItems) {
					item.batchOfInputFrames.clear(); // done with batch storage
					item.numVehiclesInferred = 0;
					item.vehicleDetectionDone = true;
					item.vehicleAttributesDetectionDone = false;
					pipeS1toS2Fifo.push(item);
				}
        	}
```


#### Pipeline Stage #2: Start Inferring Vehicle Attributes

Stage #1 from Tutorial Step 3 does both submit and wait for a inference request.  For asynchronous operation, it is broken up into Stage #2 to do the submit and Stage #3 to do the wait.  When VehicleAttribs.enabled() is false, Stage #2 has nothing to do so simply passes input frame to Stage #3. When VehicleAttribs.enabled() is true, the new Stage #2 looks similar to before but now has to handle submitting requests asynchronously which may require multiple passes for an input frame.

1. If Vehicle attributes model is enabled, then if there is an input frame coming from Stage #1 and requests are available to submit requests, then process an input frame:

```Cpp
			if (VehicleAttribs.enabled()) {
				if (!pipeS1toS2Fifo.empty() && VehicleAttribs.canSubmitRequest()) {
```


2. Grab a reference to the FIFO first item which will remain in the FIFO until all inference requests for the vehicles found have been made.  This can take multiple passes through Stage #2 because of batch size and/or the number of requests that can be made simultaneously. 

```cpp
					// grab reference to first item in FIFO, but do not pop until done inferring all vehicles in it
					FramePipelineFifoItem& ps1s2i = pipeS1toS2Fifo.front();

					const int totalVehicles = ps1s2i.vehicleLocations.size();
```


3. Enqueue vehicles up to batch limit for inference starting where left off at "numVehiclesInferred":

```Cpp
					// enqueue input batch
					for(int rib = ps1s2i.numVehiclesInferred; rib < totalVehicles; rib++) {
						if (VehicleAttribs.enquedVehicles >= VehicleAttribs.maxBatch) {
							break;
						}
						auto clippedRect = ps1s2i.vehicleLocations[rib] & cv::Rect(0, 0, width, height);
						auto Vehicle = (*ps1s2i.outputFrame)(clippedRect);
						VehicleAttribs.enqueue(Vehicle);
					}
```


4. If there are vehicles, then submit an inference request recording number of inferences now started:

```Cpp
					// ----------------------------Run vehicleResult attribute inference ----------------
					// if there are vehicles to infer, submit a request to start
					if (VehicleAttribs.enquedVehicles > 0) {
						// track how many vehicles have been inferred
						ps1s2i.numVehiclesInferred += VehicleAttribs.enquedVehicles;
						AttribsInferred += VehicleAttribs.enquedVehicles;

						t0 = std::chrono::high_resolution_clock::now();
						VehicleAttribs.submitRequest();

					}
```


5. If this input frame has submitted inference requests for all its vehicles, then mark it complete by removing (pop’ing) it from the input FIFO.  Whether all inferences are complete or not, always send an input frame to Stage #3 so it can look for inference results.

```cpp
					// make a copy before sending out
					FramePipelineFifoItem ps2s3out(ps1s2i);
					if ( ps2s3out.numVehiclesInferred >= totalVehicles) {
						// done with input FIFO item, pop from input FIFO
						pipeS1toS2Fifo.pop();
					}
					// always queue frame data for next pipeline stage to handle results even without doing inference
					pipeS2toS3Fifo.push(ps2s3out);
				}
```


6. VehicleAttribs.enabled() is false, just pass through the input frames:

```Cpp
			} else {
				// not running vehicle attributes, just pass along frames
                                               // ... pass data from pipeS1toS2Fifo to pipeS2toS3Fifo ...
			}
```


#### Pipeline Stage #3: Process Vehicle Attribute

Stage #3 is responsible for checking for and then processing vehicle attributes inference results started by Stage #2.  When VehicleAttribs.enabled() is false, Stage #3 has nothing to do so simply passes input frame to Stage #4. When VehicleAttribs.enabled() is true, Stage #3 looks similar to how results were processed, but now has to handle results asynchronously and may require multiple passes to complete an input frame.

1. Vehicle attributes model is enabled, then if there is an input frame coming from Stage #2, then process an input frame:

```Cpp
			/* *** Pipeline Stage 3: Process Vehicle Attribute Inference Results *** */
			if (VehicleAttribs.enabled()) {
				if (!pipeS2toS3Fifo.empty()) {
```


2. Grab the first input frame in the FIFO but do not remove it until it has actually been processed which may not happen if inference results are not ready.

```Cpp
					FramePipelineFifoItem ps2s3i = pipeS2toS3Fifo.front();
					int numVehicles = ps2s3i.vehicleLocations.size();
```


3. If the input frame has no vehicles, then remove it from FIFO and send to Stage #4.

```cpp
					if ( 0 == numVehicles) {
						// no vehicles are being inferred for this frame, we are done with it
						pipeS2toS3Fifo.pop();
						// queue frame data for next pipeline stage to handle results
						ps2s3i.vehicleAttributesDetectionDone = true;
						pipeS3toS4Fifo.push(ps2s3i);
					} else {
```


4. Input frame has vehicle(s) to infer.  If this is the first input frame (accumVehAttribsIsEmpty==true) then reset the attribute accumulator to the new input frame, otherwise results will be added to the current input frame "accumVehAttribs".

```Cpp
						// expecting inference results, check for them while accumulating results
						// if first FIFO item for results, initialize from first item
						if (accumVehAttribsIsEmpty) {
							accumVehAttribs = ps2s3i;
							accumVehAttribsIsEmpty = false;
						}
```


5. First is to check whether there is work to be done.  When running synchronously (!runningAsync) if there is a request in progress, then enter the stage to wait for results.  When running asynchronously, check to see if a result is ready, then enter the stage and wait for the result (which will be a short wait).

```Cpp
						if ((!runningAsync && VehicleAttribs.requestsInProcess()) || VehicleAttribs.resultIsReady()) {
							// wait for results, async will be ready
							VehicleAttribs.wait();
							t1 = std::chrono::high_resolution_clock::now();
							AttribsNetworkTime += std::chrono::duration_cast<ms>(t1 - t0);
```


6. Fetch results which will be put into VehicleAttribs.results:

```Cpp
							// ----------------------------Process outputs-----------------------------------------------------
							VehicleAttribs.fetchResults();
```


7. Store results in current input frame in "accumVehAttribs"

```Cpp
							int numVAResuls = VehicleAttribs.results.size();

							int batchIndex = 0;
							while(batchIndex < numVAResuls) {
								VehicleAttribsDetection::Attributes& res = VehicleAttribs.results[batchIndex];
								accumVehAttribs.vehicleAttributes.push_back(res);
								batchIndex++;
							}
```


8. Track number of inferred results.  If inference of all vehicles has been completed, then send off input frame to Stage #4 and mark accumVehAttribs as empty:

```cpp
							accumVehAttribs.numVehiclesInferred = ps2s3i.numVehiclesInferred;

							// only send out frame data when inference is complete
							if ( accumVehAttribs.numVehiclesInferred >= numVehicles) {
								// queue frame data for next pipeline stage to handle results
								accumVehAttribs.vehicleAttributesDetectionDone = true;
								pipeS3toS4Fifo.push(accumVehAttribs);
								accumVehAttribsIsEmpty = true;
							}
```


9. The inference results were processed for the input frame in the FIFO, remove it form the FIFO:

```Cpp
							// done with this FIFO item
							pipeS2toS3Fifo.pop();
						}
					}
				}
```


10. VehicleAttribs.enabled() is false, just pass through the input frames:

```Cpp
			} else {
				// not running vehicle attributes, just pass along frames
                                               // ... pass data from pipeS2toS3Fifo to pipeS3toS4Fifo ...
			}
```


#### Pipeline Stage #4: Render Results

The last pipeline stage renders all the results and looks just like the last stage in Tutorial Step 3 with only small changes being:

1. The stage now reads from Stage #3 via pipeS3toS4Fifo:

```Cpp
			if (!pipeS3toS4Fifo.empty()) {
				FramePipelineFifoItem ps3s4i = pipeS3toS4Fifo.front();
				pipeS3toS4Fifo.pop();
```


2. When running asynchronously, the timing statistics are not accurate and are skipped:

```Cpp
				// When running asynchronously, timing metrics are not accurate so do not display them
				if (!runningAsync) {
					out.str("");
					out << "Vehicle detection time ";
					// .. statistics output code ...
```


#### End of Loop

Finally, the end of the main loop now checks to see that all stages are done before allowed to end.  The "done" variable now checks to make sure all stage FIFOs are empty before indicating the loop is done.

```Cpp
            // wait until break from key press after all pipeline stages have completed
            done = !haveMoreFrames && pipeS0toS1Fifo.empty() && pipeS1toS2Fifo.empty() && pipeS2toS3Fifo.empty() && pipeS3toS4Fifo.empty();
```


# Building and Running

Now let us build and run the complete application and see how it runs all three analysis models.

## Build

1. Open up an Xterm window or use an existing window to get to a command shell prompt.

2. Change to the directory containing Tutorial Step 4:

```bash
cd tutorials/car_detection_tutorial/step_4
```


3. The first step is to configure the build environment for the OpenCV toolkit by sourcing the "setupvars.sh" script.

```bash
source  /opt/intel/computer_vision_sdk/bin/setupvars.sh
```


4. Now we need to create a directory to build the tutorial in and change to it.

```bash
mkdir build
cd build
```


5. The last thing we need to do before compiling is to configure the build settings and build the executable.  We do this by running CMake to set the build target and file locations.  Then we run Make to build the executable.

```bash
cmake -DCMAKE_BUILD_TYPE=Release ../
make
```


## Run

1. Before running, be sure to source the helper script that will make it easier to use environment variables instead of long names to the models:

```bash
source ../../scripts/setupenv.sh 
```


2. First, let us see how it works on a single image file using default synchronous mode.

```bash
./intel64/Release/car_detection_tutorial -m $mVLP32 -m_va $mVA32 -i ../../data/car_1.bmp
```


3. The output window will show the image overlaid with colored rectangles over the cars and license plate along with and the timing statistics for computing the results.  Now run the command again in asynchronous mode using the option "-n_async 2":

```bash
./intel64/Release/car_detection_tutorial -m $mVLP32 -m_va $mVA32 -i ../../data/car_1.bmp -n_async 2
```


4. The performance should be the same because a single image was run which is effectively the same as running synchronously since each pipeline stage must wait for the one image to process.

5. Next, let us try it on a video file. 

```bash
./intel64/Release/car_detection_tutorial -m $mVLP32 -m_va $mVA32 -i ../../data/car-detection.mp4 -n_async 1
```


6. Over each frame of the video, you will see green rectangles drawn around the cars as they move through the parking lot.  Now run the command again in asynchronous mode using the option "-n_async 2":

```bash
./intel64/Release/car_detection_tutorial -m $mVLP32 -m_va $mVA32 -i ../../data/car-detection.mp4 -n_async 2
```


7. Unexpectedly, asynchronous mode should have made the video take longer to run by >10%.  Why would that be?  With the main loop now running asynchronously and not blocking, it is now an additional thread running on the CPU along with the inference models.  Now let us shift running the models to other devices, first in synchronous mode then asynchronous with increasing -n_async value using the commands:

```Bash
./intel64/Release/car_detection_tutorial -m $mVLP16 -m_va $mVA16 -d MYRIAD -d GPU -i ../../data/car-detection.mp4 -n_async 1
./intel64/Release/car_detection_tutorial -m $mVLP16 -m_va $mVA16 -d MYRIAD -d GPU -i ../../data/car-detection.mp4 -n_async 2
./intel64/Release/car_detection_tutorial -m $mVLP16 -m_va $mVA16 -d MYRIAD -d GPU -i ../../data/car-detection.mp4 -n_async 4
./intel64/Release/car_detection_tutorial -m $mVLP16 -m_va $mVA16 -d MYRIAD -d GPU -i ../../data/car-detection.mp4 -n_async 8
```


8. Asynchronous mode should be faster by some amount for "-n_async 2" then a little more for “-n_async 4”, then not really noticeable fo “-n_async 8”.  The improvements come from the CPU running in parallel more and more with the GPU and Myriad.  The absence of improvement shows when the CPU is doing less in parallel and is waiting on the other devices.  This is referred to as “diminishing returns” and will vary across devices and inference models.

9. User exercise: Modify the last commands used to try different combinations of GPU and Myriad to find the fastest asynchronous combination for your hardware.

    1. Hint: You may need to increase well beyond  "-n_async 8" to hit the point of diminishing returns.

# Conclusion

By adding the asynchronous use of the Inference Engine API we have seen how it can affect performance.  Performance improves when offloading the models to other devices leaving the CPU to do all the data preparation and image rendering work in parallel.  However, there is a limit to performance improvement when the CPU (or other) device starts to wait on its parallel partner devices.

# Navigation

[Car Detection Tutorial](../Readme.md)

[Car Detection Tutorial Step 3](../step_3/Readme.md)
