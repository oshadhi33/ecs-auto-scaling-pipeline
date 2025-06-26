pipeline {
    agent any  // Run the pipeline on any available Jenkins agent

    environment {
        // Set the default AWS region to use in all stages
        AWS_DEFAULT_REGION = 'us-east-1'
    }

    stages {
        stage('Setup Python venv') {
            steps {
                // Create a Python virtual environment and install required Python packages
                sh '''
                    python3 -m venv venv             // Create a virtual environment named 'venv'
                    . venv/bin/activate              // Activate the virtual environment
                    pip install --upgrade pip        // Upgrade pip to the latest version
                    pip install boto3                // Install boto3 AWS SDK for Python
                '''
            }
        }

        stage('Prepare AWS Credentials') {
            steps {
                // Inject AWS credentials securely from Jenkins credentials store
                withCredentials([
                    [$class: 'UsernamePasswordMultiBinding',
                     credentialsId: 'Your credentials ID',  // Jenkins credentials ID
                     usernameVariable: 'AWS_ACCESS_KEY_ID',                  // Map username to AWS_ACCESS_KEY_ID env var
                     passwordVariable: 'AWS_SECRET_ACCESS_KEY']              // Map password to AWS_SECRET_ACCESS_KEY env var
                ]) {
                    // Just print a confirmation message that credentials are loaded
                    sh 'echo "AWS credentials loaded"'
                }
            }
        }

        stage('Get All ECS Clusters') {
            steps {
                // Write a Python script to list all ECS clusters in the AWS region
                writeFile file: 'get_clusters.py', text: """\
import boto3

ecs = boto3.client('ecs')

def get_all_clusters():
    clusters = []
    paginator = ecs.get_paginator('list_clusters')  // Use paginator to handle large result sets
    for page in paginator.paginate():
        clusters.extend(page['clusterArns'])         // Append cluster ARNs from each page
    print("\\n".join(clusters))                      // Output cluster ARNs, one per line

if __name__ == "__main__":
    get_all_clusters()
"""

                // Run the Python script in the virtual environment and save output to clusters.txt
                sh '''
                    . venv/bin/activate
                    python3 get_clusters.py > clusters.txt
                '''
                // Read clusters.txt into a Groovy list and print it to Jenkins console
                script {
                    def clusters = readFile('clusters.txt').trim().split("\n")
                    echo "Clusters found: ${clusters}"
                }
            }
        }

        stage('Check and Scale Services') {
            steps {
                // Write a Python script that checks ECS service health and scales ASGs if needed
                writeFile file: 'check_and_scale.py', text: """\
import boto3
from botocore.exceptions import ClientError

ecs = boto3.client('ecs')
asg = boto3.client('autoscaling')

KNOWN_ISSUES = {
    "RESOURCE:MEMORY": "Insufficient memory in capacity provider",
    "RESOURCE:CPU": "Insufficient CPU in capacity provider"
}

def get_services(cluster):
    service_arns = []
    paginator = ecs.get_paginator('list_services')        // Paginate through services for the cluster
    for page in paginator.paginate(cluster=cluster):
        service_arns.extend(page['serviceArns'])         // Collect service ARNs
    return service_arns

def check_service_health(cluster, service_arn):
    response = ecs.describe_services(cluster=cluster, services=[service_arn])
    service = response['services'][0]

    // Extract capacity providers from the service's capacity provider strategy, if any
    capacity_providers = [cp['capacityProvider'] for cp in service.get('capacityProviderStrategy', [])]
    
    if not capacity_providers:
        // If no capacity providers assigned, skip scaling logic
        return 'skip', service, None
    
    cp_name = capacity_providers[0]
    running_count = service['runningCount']           // Number of running tasks
    desired_count = service['desiredCount']           // Desired number of tasks

    print(f"\\nChecking {service['serviceName']} in {cluster}: Running={running_count}, Desired={desired_count}")
    
    // If running tasks are fewer than desired, mark service as unhealthy
    if running_count < desired_count:
        return 'unhealthy', service, cp_name
    
    return 'healthy', service, cp_name

def scale_up_asg_from_capacity_provider(cp_name):
    try:
        // Get capacity provider details including linked ASG ARN
        response = ecs.describe_capacity_providers(capacityProviders=[cp_name])
        cp = response['capacityProviders'][0]

        // Extract Auto Scaling Group name from the ASG ARN
        asg_name = cp['autoScalingGroupProvider']['autoScalingGroupArn'].split('/')[-1]

        // Describe the ASG to get current capacity details
        response = asg.describe_auto_scaling_groups(AutoScalingGroupNames=[asg_name])
        group = response['AutoScalingGroups'][0]

        desired = group['DesiredCapacity']
        max_size = group['MaxSize']

        // Increase desired capacity by 1 if below max size
        if desired < max_size:
            new_desired = desired + 1
            print(f"Scaling ASG {asg_name}: {desired} → {new_desired}")
            asg.update_auto_scaling_group(AutoScalingGroupName=asg_name, DesiredCapacity=new_desired)
        else:
            print(f"ASG {asg_name} already at max size ({max_size})")
    except Exception as e:
        print(f"Error scaling ASG from capacity provider {cp_name}: {e}")

if __name__ == "__main__":
    // Read cluster ARNs from clusters.txt
    with open('clusters.txt') as f:
        clusters = [line.strip() for line in f if line.strip()]
    
    // For each cluster, get services and check health
    for cluster in clusters:
        services = get_services(cluster)
        for service_arn in services:
            status, service, cp_name = check_service_health(cluster, service_arn)
            // Scale up ASG if service is unhealthy and capacity provider exists
            if status == 'unhealthy' and cp_name:
                print(f"Scaling up capacity provider: {cp_name}")
                scale_up_asg_from_capacity_provider(cp_name)
"""

                // Run the Python health check and scaling script inside the virtual environment
                sh '''
                    . venv/bin/activate
                    python3 check_and_scale.py
                '''
            }
        }
    }

    post {
        failure {
            // On pipeline failure, print an error message with possible causes
            echo 'Pipeline failed. Please check the logs and verify AWS credentials and permissions.'
        }
    }
}
