# XRECO 4.2 Neural Rendering Services: GDGS
XRECO

DAI, MAIP, 2024

CODEOWNER: sergio.montoya@i2cat.net

---

PyTorch implementation of GDGS: Generalizable Depth-based Gaussian Splatting for sparse view synthesis. GDGS was developed under the XRECO Project. XRECO is an HorizonEurope Innovation Project co-financed by the EC under Grant Agreement ID: 101070250. 

GDGS renders real-time views on unseen scenes from few views using a rasterizer of 3DGS. GDGS is a new real-time generalizable view synthesis model capable of generating high-quality views even with very few input views that are far apart from each other. Given a set of sparse input source views, our model can generalize to new scenes and generate highly realistic renders.
## Getting started

### Application overview

This application provides a self-contained API that processes requests and distributes tasks in a containerized environment. Specifically, it contains the following containers:

- **api_app**: FastAPI-based API that produces queries and obtains results from asynchronous tasks. This acts as an interface to the source code of GDGS.
- **celery_worker**: Celery container that creates asynchronous tasks. This is the main component which runs the code of GDGS as asynchronous tasks.
- **redis**: Message broker and database for the queue and jobs. It manages the communication between the API and the Celery container.

#### Creating a configuration file

For defining the configuration parameters, an env file with the name inf_server_config.env has to be created. The initial parameters are loaded from this file when the Docker image and docker-compose are started.

Description:

| **Setting parameter** | **Description** |
| --- | --- |
| VERSION | Version of the setting parameters. |
| SERVICE_PORT | Port of the API service. |
| REMOTE_DATA_TYPE | Data URLs point to AWS, MinIO or a general URL that can be downloaded with wget: "AWS", "minio" or "url" |
| MINIO_USER | Username for the MinIO storage.   |
| MINIO_PASSWORD | Password for the MINIO storage. |
| MINIO_ENDPOINT | MinIO base URL. |
| AWS_REGION | Region for the AWS S3 storage.   |
| AWS_USER | Username for the AWS S3 storage. |
| AWS_PASSWORD | Password for the AWS S3 storage. |

#### Start containers

First clone and set this repository as the workspace for locating necessary files:
```
git clone git@github.com:XRecoEU/WP4-T4.2-Neural_Rendering_Services.git
cd WP4-T4.2-Neural_Rendering_Services/
export workspace=$(pwd)
# Initialize and clone submodules
git submodule init
git submodule update
```

For a step by step example, follow the guide that is on the bottom part of this README.md file.

When you want to start all the containers, you can run the following command:
```
docker compose --env-file inf_server_config.env up -d
```

Access the FastAPI service at port **8000**.
See the available endpoint services of FastAPI by accessing to **/docs**.

## API Summary
### Endpoints

- /upload_zip/{dataset_name}: Upload a dataset to the dataset volume on the docker container.
- /train: Starts finetuning the generalized model on the specified dataset.
- /download_experiment/{experiment_name}: Downloads an RGB and depth videos of new views from different viewpoints from a finetuned model. It also downloads the model weights.
- /check_status/{experiment_name}: Checks the status of a finetuning experiment.
- /infer_views_from_zip: Synthesizes a video of a new camera trajectory of a seen or unseen scene.

### Workflow

<ol>
  <li>Upload data to an AWS container and set the according credentials in the configuration file `inf_server_config.env`.</li>
  <li>Download the data with `/upload_zip/{dataset_name}` and start finetuning on a specific scene with `/train`.</li>
  <li>Check the status of the finetuning experiment to see the progress `/check_status/{experiment_name}`.</li>
  <li>When the training has finished, qualitative examples of the trained model and its weights can be obtained with `/download_experiment/{experiment_name}`.</li>
  <li>Once the generalized model is finetuned on a scene, it can generate new views of seen or unseen time instants with `/infer_views_from_zip`. The file structure for the inference is specified in the corresponding API endpoint. </li>
</ol>



## API Endpoints

<b> /upload_zip/{dataset_name} </b>
The API also requires the dataset to be uploaded with the following structure to an AWS S3 container:

```
dataset.zip
 -- manifest.json
 -- asset
    -- calibration.json
    -- depth
        -- depth-000-000000.exr
        -- depth-000-000001.exr
        -- depth-000-000002.exr
        ...
    -- rgb
        -- image-000-000000.jpg
        -- image-000-000001.jpg
        -- image-000-000002.jpg
        ...
```

Where the calibration file has the following structure:

```
{
    "extrinsics": [],
    "intrinsics": [],
    "distortion": []
}
```

### Train
<b> /train </b>

Finetunes GDGS in a specific scene, which has to be a dataset which has been previously uploaded with <b> /upload_zip/{dataset_name} </b>. 

<details>
    <summary> API call Body content example </summary>

```json
{
  "data": {
    "experiment_name": "<experiment name>",
    "downsize_factor": 8,
    "output_quality": "<select one: accurate OR balanced>",
    "dataset_name": "<dataset name>"
  }
}
```

</details>

### Download experiment
<b> /download_experiment/{experiment_name} </b>: Downloads RGB and depth video examples along with the weights of the model.

### Check training status
<b> /check_status/{task_id} </b>: Given the task id obtained by submitting a training, obtain the status of the task.

### Infer new view / video
<b> /infer_views_from_zip </b>: Synthesize a new camera trajectory given a set of videos and camera parameters.
The zip has the following structure:

```
dataset.zip
 -- manifest.json
 -- asset
    -- calibration.json
    -- depth-000.mkv
    -- depth-001.mkv
    -- depth-002.mkv
    -- image-000.mp4
    -- image-001.mp4
    -- image-002.mp4
    ...
```

- The depth .mkv files are uint16 mkv videos with scaled depth at millimeters
- The calibration.json contains the list of input cameras and target camera trajectory:
```
{
    "extrinsics": [],
    "intrinsics": [],
    "distortion": [],
    "target_camera": []
}
```

## Step by step AWS example

When you want to start all the containers, you can run the following command:
```
docker compose --env-file inf_server_config.env up -d
```
1. Connect to the FastAPI endpoints interface by accessing to the address `localhost:8000/docs` with any web browser. If this is running on a remote server, substitute localhost with the actual IP address.
2. Call the `/upload_zip/{dataset_name}` endpoint by specifying a dataset name, which for our example we will use `Bet_API_train.zip`. You can check the status of the downloading task by calling the `/check_status/{task_id}` endpoint and providing the `task_id` returned by calling the upload task.
```
dataset_name: API_Bet_ORBEC

{
  "zip_url": "https://xreco-nmr.s3.eu-west-1.amazonaws.com/gdnerf_test/Bet_API_train.zip",
  "invert_extrinsics": true
}
```

3. Once the dataset has been downloaded locally, we can start the finetuning training. For doing so, we will use the `/train` endpoint. Provide the following Request body:

```
{
  "data": {
    "experiment_name": "Bet_orbec_test",
    "downsize_factor": 4,
    "output_quality": "balanced",
    "dataset_name": "API_Bet_Orbec"
  }
}
```
- The `downsize_factor` is the downsampling factor applied to input data, as big resolution images might result in out of memory CUDA errors in the Celery workers.
- The `dataset_name` must be the one assigned to the dataset.

4. Again, the `/train` will start the training and will return a `task_id`, which can be used in the `/check_status/{task_id}` call to see the progress of the finetuning step.
5. Once it has been trained, we can see some example color and depth views in video format, along with the network weights by calling the `/download_experiment/{experiment_name}` endpoint. We have to specify the `experiment_name` to download it.
6. In order to infer new views from few images or videos of a finetuned scene or a new unseen scene, we provide the `/infer_views_from_zip` endpoint. We provide an example that interpolates among the available cameras of the `Bet_API_train.zip` dataset. The example interpolates the camera among the three available views:

```
{
  "params": {
    "zip_url": "https://xreco-nmr.s3.eu-west-1.amazonaws.com/gdgs_test/video_inference_Bet.zip",
    "experiment_name": "Bet_orbec_test",
    "downsize_factor": 2,
    "depth_scaling": 1.0,
    "invert_extrinsics": true
  }
}
```

## Resources

- [Input example ZIP](https://drive.google.com/file/d/1DFlu2_hcWa2XeLLCVptxak4F1mZD52Dd/view?usp=drive_link)
- [View synthesis example ZIP](https://drive.google.com/file/d/1eYZvMjhWlTvi9dPCXbIDF6l-Q48d9Z_I/view?usp=sharing)
- [calibration.json example](https://drive.google.com/file/d/1pFS-ogEszsHOmXJx5P1MVq6tmM0MYUE0/view?usp=sharing)
- [Video result output]() # TODO

## API usage video example

# TODO

## Acknowledgement

This project uses [REDIS](https://redis.com/) as broker and database, which is distributed under the [BSD-3-Clause](https://opensource.org/license/bsd-3-clause/). A list of python packages used and their licenses can be found at `third_party_licenses.txt`.

## License

This code has been developed by Fundació Privada Internet i Innovació Digital a Catalunya (i2CAT). i2CAT is a *non-profit research and innovation centre* that  promotes mission-driven knowledge to solve business challenges, co-create solutions with a transformative impact, empower citizens through open and participative digital social innovation with territorial capillarity, and promote pioneering and strategic initiatives. i2CAT *aims to transfer* research project results to private companies in order to create social and economic impact via the out-licensing of intellectual property and the creation of spin-offs. Find more information of i2CAT projects and IP rights at https://i2cat.net/tech-transfer/

This code is licensed under the terms of XRECO consortium agreement with limited access for project implementation. Access to the source code does not imply authorization to use it for other purposes. XReco is a Horizon Europe Innovation Project co-financed by the EC under Grant Agreement ID: 101070250.

If you find that this license doesn't fit with your requirements regarding the use, distribution or redistribution of our code for your specific work, please, don’t hesitate to contact the intellectual property managers in i2CAT at the following address: techtransfer@i2cat.net
