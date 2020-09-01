
pipeline {
   agent { label 'node1' }
   
   environment { 
	   
	DOCKER_IMAGE = 'nodejs/app'
	   
	ECR_REPO = '007293158826.dkr.ecr.ap-southeast-1.amazonaws.com/nodejs'
	APP_VERSION = "${BUILD_ID}"
        APP_ENV = "${BRANCH_NAME}"
       
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
	AWS_DEFAULT_REGION    = 'ap-southeast-1'
	AWS_DEFAULT_OUTPUT    = 'json'
	   
	STAGING_TASK    = 'nodejs-staging-task'
	STAGING_CLUSTER = 'nodejs-staging-cluster'
	STAGING_SERVICE = 'nodejs-staging-srv'
	   
	RELEASE_TASK    = 'nodejs-release-task'
	RELEASE_CLUSTER = 'nodejs-release-cluster'
	RELEASE_SERVICE = 'nodejs-release-srv'
   }

   stages {

      stage('[NODEJS] Build') {
         steps {
            echo '****** Build app ******'
            sh '''
            docker build -t ${DOCKER_IMAGE}:${BUILD_ID} .
            docker tag ${DOCKER_IMAGE}:${BUILD_ID} ${ECR_REPO}:${BUILD_ID}
            '''
         }
      }
      
      stage('[NODEJS] Push to ECR') {
         steps {
            echo '****** Push docker image to ECR ******'
	    sh '''
            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
	    export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
            export AWS_DEFAULT_OUTPUT=${AWS_DEFAULT_OUTPUT}
            
            ECR_LOGIN_STRING=`aws ecr get-login --region ${AWS_DEFAULT_REGION} --no-include-email`
            
            eval ${ECR_LOGIN_STRING}
            docker push ${ECR_REPO}:${BUILD_ID}
            '''
         }
      }
      
      stage('[NODEJS] Deploy to staging') {
            when {
                branch 'staging' 
	    }
            steps {
		echo "****** Deploy to ${BRANCH_NAME} branch ******"
		sh '''
		TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition "${STAGING_TASK}")
		NEW_TASK_DEFINITION=$(echo $TASK_DEFINITION | jq --arg ECR_REPO "${ECR_REPO}:${APP_VERSION}" --arg APP_VERSION "${APP_VERSION}" --arg APP_ENV "${APP_ENV}" '.taskDefinition |.containerDefinitions[0].image = $ECR_REPO | .containerDefinitions[0].environment[0].value = $APP_VERSION |.containerDefinitions[0].environment[1].value = $APP_ENV  | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities)')
		TASK_VERSION=$(aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEFINITION" | jq --raw-output '.taskDefinition.revision')
 		ECS_UPDATE=$(aws ecs update-service --force-new-deployment --cluster ${STAGING_CLUSTER}  --service ${STAGING_SERVICE} --task-definition ${STAGING_TASK}:$TASK_VERSION  | jq --raw-output '.service.serviceName')
		'''
            }
        }
      stage('[NODEJS] Deploy to production') {
           when {
                branch 'release' 
            }
            steps {
		echo "****** Deploy to ${BRANCH_NAME} branch ******"
		sh '''
		TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition "${RELEASE_TASK}")
		NEW_TASK_DEFINITION=$(echo $TASK_DEFINITION | jq --arg ECR_REPO "${ECR_REPO}:${APP_VERSION}" --arg APP_VERSION "${APP_VERSION}" --arg APP_ENV "${APP_ENV}" '.taskDefinition |.containerDefinitions[0].image = $ECR_REPO | .containerDefinitions[0].environment[0].value = $APP_VERSION |.containerDefinitions[0].environment[1].value = $APP_ENV  | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities)')
		TASK_VERSION=$(aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEFINITION" | jq --raw-output '.taskDefinition.revision')
 		ECS_UPDATE=$(aws ecs update-service --force-new-deployment --cluster ${RELEASE_CLUSTER}  --service ${RELEASE_SERVICE} --task-definition ${RELEASE_TASK}:$TASK_VERSION  | jq --raw-output '.service.serviceName')
		'''
            }
        }
   }
}
