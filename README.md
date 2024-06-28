# Operationalizing ML using AWS Sagemaker, EC2 and Lambda
This project contains a set of files and examples to:

1) Train a model using the AWS Sagemaker tool
2) Deploy the model to an AWS Endpoint
3) Train the same model but using AWS EC2 only
4) Trigger inference using a Lambda function
5) Configuring Concurrency and Auto scaling for the Lambda and deployed endpoint.

### Notebook Instance Creation
As a first step, we create a new Notebook in Sagemaker, the example file is `train_and_deploy_solution.ipynb`. I've selected an `ml.m5.xlarge` instance type given that its cost is relatively low ($0.23 per hour) and it provides 4vCPU, which should be more than enough for our purposes. Note that this will only be used to run the notebook, for training we will be using a more powerful instance.

<img width="1370" alt="Screenshot 2024-06-26 at 6 09 48 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/442b81e9-f4a7-4ad9-8437-5cb51b89674b">

### Downloading data to S3 Bucket
The first cells in the notebook download the training and test data to our AWS workspace. We download the images and copy them to an S3 bucket.

<img width="1368" alt="Screenshot 2024-06-27 at 5 21 01 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/321e8bcf-855e-4710-80ee-520fce008db1">

### Hyperparameter tuning
We first train our model testing out different combinations of hyperparameters, until we find out the best combination of them.

<img width="995" alt="Screenshot 2024-06-27 at 5 24 34 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/827e2764-71b5-4e9e-8beb-4c4e4735ad5e">

## Single Instance Training
Once we've found the best `Estimator`, we perform single instance training.

<img width="1327" alt="Screenshot 2024-06-27 at 5 25 46 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/6a2d9c00-e56d-4ec6-a459-99e087a8ef57">

## Multi-Instance Training
Then we use the same `Estimator`, but running on multiple instances.

<img width="1347" alt="Screenshot 2024-06-27 at 5 26 21 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/8181c09c-78e4-4fca-906f-60e35fe400e8">

## Deployment
Finally, we take the trained model and deploy it to an AWS Endpoint so that we can easily perform inference on it

<img width="1357" alt="Screenshot 2024-06-27 at 5 27 07 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/affa350f-c34c-4586-88ab-8b375e573bb4">

## EC2
The same functionality obtained using Sagemaker can be leveraged using EC2. We won't be able to use the Sagemaker libraries, but there should be a cost reduction. 
For this, we launch a new EC2 instance. In this case, I've used the AWS Deep Learning AMI. I've selected a `G4dn.xlarge` instance, which is the smallest possible for this AMI.

<img width="1252" alt="Screenshot 2024-06-28 at 5 24 04 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/4d440281-617d-4d56-9dde-a344bca865ee">

The code to perform training on EC2 is located in `ec2train1.py`. The script is similar to the Sagemaker one, with a few differences:

1. Argument Parsing and Environment Variables
SageMaker: Uses argparse to parse command-line arguments for hyperparameters (learning rate, batch size) and paths for data and model directories. EC2: Hard-codes the values for batch size, learning rate, and data paths directly within the script. No command-line argument parsing is used.

2. Main Function and Script Execution
SageMaker: Defines a main function that is called when the script is executed. This function handles argument parsing, logging, data loader creation, model training, testing, and saving. On EC2 the script executes sequentially from top to bottom.

The main differences between the EC2 and SageMaker scripts revolve around the handling of hyperparameters, logging, and script structure to accommodate the managed environment of SageMaker. SageMaker's script is designed for scalability and flexibility, allowing for easy changes and better integration with SageMaker's managed services, while the EC2 script is more straightforward and hard-coded for a specific setup.

The main advantage of EC2 is that its instances can be scaled up or down easily. Cost should be lower and all capabilities (Number of CPU, GPU, Memory, Disk) can be easily customized by selecting the appropriate instance. 

Sagemaker on the other hand has the advantage of the flexibility it provides, quick set-up, and wide range of libraries. It should be a better option for quick experimentation.

Final output of EC2 training:

<img width="583" alt="Screenshot 2024-06-27 at 7 37 30 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/a47ba5b0-2410-4a61-9a71-5433a75d22e1">

## Lambda
The main benefit of AWS Lambda is that it allows you to run code in response to events without provisioning or managing servers.
We define a new AWS Lambda function to handle incoming events, invoking a SageMaker endpoint with the event data, and return the processed response. In this case, we will use the endpoint created previously.

The code for the lambda function can be found in `lambdafunction.py`.

<img width="1549" alt="Screenshot 2024-06-28 at 5 46 12 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/3448d0fa-8ef1-4aa6-9092-1536213901f8">

## IAM Permissions
To access the Sagemaker endpoint, we need to grant the Lambda function the necessary permissions. For this, we create a new policy giving Sagemaker full access and attach it to our Lambda function role.

<img width="1155" alt="Screenshot 2024-06-28 at 4 25 13 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/0a42770d-a634-4654-b70e-136df7322e5b">

This is just an experiment, but in production environments, we'd want to avoid giving full access unless it is absolutely necessary, and we should only define the minimum required policies. 

## Testing after permission change

<img width="1326" alt="Screenshot 2024-06-28 at 4 27 14 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/b2459a75-e0b8-4746-b139-24b7e136975a">

```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "text/plain",
    "Access-Control-Allow-Origin": "*"
  },
  "type-result": "<class 'str'>",
  "COntent-Type-In": "LambdaContext([aws_request_id=60de42f8-8ba7-42bb-ab81-61a9cced3ccb,log_group_name=/aws/lambda/mlopslambdafn,log_stream_name=2024/06/28/[$LATEST]00f886587d304b86b572d6318e7a5075,function_name=mlopslambdafn,memory_limit_in_mb=128,function_version=$LATEST,invoked_function_arn=arn:aws:lambda:us-east-1:420772541149:function:mlopslambdafn,client_context=None,identity=CognitoIdentity([cognito_identity_id=None,cognito_identity_pool_id=None])])",
  "body": "[[0.07954253256320953, 0.40282300114631653, 0.07961469888687134, 0.12384890019893646, 0.6003313660621643, 0.2056613713502884, 0.2630038857460022, -0.07065707445144653, -0.18416281044483185, -0.2060999721288681, 0.34379053115844727, 0.5675066709518433, -0.13721860945224762, 0.11116041243076324, 0.19649425148963928, 0.3022119104862213, 0.23152992129325867, -0.2233394831418991, 0.20486512780189514, 0.33096233010292053, 0.782687783241272, -0.01337601151317358, 0.15035845339298248, 0.008738936856389046, -0.6942951083183289, -0.38775765895843506, 0.255708783864975, -0.48628759384155273, 0.35866692662239075, -0.1697039008140564, 0.2712080478668213, -0.17511802911758423, -0.1331639438867569, 0.21256567537784576, 0.375177800655365, 0.5992130637168884, 0.17122168838977814, -0.03490719571709633, 0.036180827766656876, -0.13472342491149902, 0.24207304418087006, 0.34049975872039795, 0.2807250916957855, -0.1305491328239441, 0.14714615046977997, 0.3261350393295288, 0.09317479282617569, 0.06458776444196701, -0.17451296746730804, 0.12563282251358032, 0.15845343470573425, 0.030955906957387924, -0.10487108677625656, 0.11414721608161926, 0.1276998221874237, 0.34390053153038025, 0.05692378059029579, -0.1447855681180954, -0.047368936240673065, 0.058568090200424194, 0.21783286333084106, 0.3229376971721649, 0.0684320330619812, -0.5833712220191956, 0.24380074441432953, -0.4958532750606537, -0.37098178267478943, 0.10782699286937714, -0.03555365651845932, -0.24999338388442993, 0.08779942989349365, -0.2537887990474701, -0.06359140574932098, -0.4443299174308777, -0.06070014834403992, 0.19551025331020355, -0.0993698239326477, 0.16962778568267822, 0.2598941922187805, 0.2613247036933899, 0.018205825239419937, 0.4306142032146454, -0.3922233283519745, 0.24951305985450745, -0.3668804168701172, 0.1904824823141098, 0.24420133233070374, -0.19070753455162048, 0.09421434998512268, 0.07738462090492249, 0.383633017539978, -0.14145085215568542, -0.12911665439605713, -0.16132646799087524, -0.12524324655532837, -0.31342485547065735, -0.1915500909090042, -0.12036626785993576, -0.30454161763191223, -0.4184434711933136, 0.3298092186450958, -0.740184485912323, 0.03514190390706062, -0.278091698884964, -0.37470975518226624, -0.08149487525224686, 0.23839770257472992, -0.3344584107398987, -0.5113279819488525, -0.3192947208881378, -0.18144352734088898, 0.09688134491443634, -0.06399595737457275, -0.5130484104156494, 0.26979729533195496, -0.3304756283760071, -0.05648273229598999, 0.24407725036144257, -0.3803858757019043, -0.18329854309558868, -0.3421548008918762, -0.31359201669692993, 0.25252822041511536, 0.11129440367221832, -0.575202465057373, -0.5984242558479309, -0.4155922532081604, -1.0165868997573853, 0.3836117684841156, -0.06853857636451721, -0.4184304475784302, -0.19241873919963837, -0.5864339470863342]]"
}

```

## Concurrency
Concurrency will make our Lambda function better able to accommodate high traffic because it will enable it to respond to multiple invocations at once. In this case, we will set it up so that up to 2 concurrent calls can happen at once. This should be more than enough for our experiment purposes. If our service suddenly becomes a high-demand one, we will start experiencing higher latencies and we should consider incrementing these values.

<img width="1368" alt="Screenshot 2024-06-28 at 4 49 01 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/91f78bfc-e128-45d1-9a8a-7cd79bf3d0fa">

## Auto-scaling
Autoscaling allows cloud resources to automatically scale up or down based on demand, optimizing performance and cost-efficiency by ensuring adequate capacity during peak times and reducing waste during low-usage periods. In this case, we are setting up autoscaling on our endpoint to 4 instances max, and we set the number of concurrent invocations (30) to be the trigger for new instance creation. 

<img width="803" alt="Screenshot 2024-06-28 at 4 50 59 PM" src="https://github.com/federicopuy/OperationalizingMLSagemaker/assets/12384264/988f1383-9c46-4630-8d9c-46a6ab174c6d">



