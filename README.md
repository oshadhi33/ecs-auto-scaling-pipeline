# ECS Auto-Scaling Jenkins Pipeline

This repository contains a Jenkins pipeline script that automates monitoring and scaling of AWS ECS services based on their health status.

---

## Overview

The pipeline performs the following tasks:

1. Sets up a Python virtual environment and installs required AWS SDK (`boto3`).
2. Loads AWS credentials securely using Jenkins credentials binding.
3. Retrieves all ECS clusters in the configured AWS region.
4. Checks the health of ECS services in each cluster.
5. If an ECS service is detected as unhealthy (running tasks less than desired), the pipeline scales up the associated Auto Scaling Group (ASG) linked to the service’s capacity provider.
6. Provides console logs for all steps and error handling for scaling actions.

---

## How to Use

1. **Jenkins Setup:**
   - Create a Jenkins pipeline job.
   - Point it to this GitHub repository.
   - Set up Jenkins credentials for AWS access (Access Key ID and Secret Access Key) and update the `credentialsId` in the Jenkinsfile accordingly.

2. **AWS Permissions:**
   - Ensure the AWS IAM user has permissions for ECS (`ecs:ListClusters`, `ecs:DescribeServices`, etc.) and Auto Scaling (`autoscaling:DescribeAutoScalingGroups`, `autoscaling:UpdateAutoScalingGroup`).

3. **Run the Pipeline:**
   - The pipeline will automatically:
     - Set up Python environment.
     - Retrieve ECS clusters.
     - Check and scale ECS services if necessary.

---

## File Details

- `Jenkinsfile` - Main declarative Jenkins pipeline script.
- Embedded Python scripts:
  - `get_clusters.py` - Fetches all ECS clusters.
  - `check_and_scale.py` - Checks ECS service health and scales ASGs if needed.

---

## Customization

- Modify AWS region by changing the `AWS_DEFAULT_REGION` environment variable in the Jenkinsfile.
- Update or add any additional Python packages in the `pip install` step as needed.
- Adjust ASG scaling logic as per company-specific policies.

---

## License

This project is provided as-is for personal portfolio use.

---

## Contact

For any questions, feel free to reach me via my GitHub profile.


