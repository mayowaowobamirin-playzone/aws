AWSTemplateFormatVersion: '2010-09-09'
Resources:
  RDSDeliverystream:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: cloudwatch-logs-rds
      S3DestinationConfiguration:
        BucketARN: 'arn:aws:s3:::cloudwatch-logs-firehose'
        BufferingHints:
          IntervalInSeconds: '60'
          SizeInMBs: '10'
        Prefix: 'aws/rds/'
        RoleARN: !GetAtt deliveryRole.Arn
        CompressionFormat: 'UNCOMPRESSED'
        EncryptionConfiguration:
          KMSEncryptionConfig:
            AWSKMSKeyARN: !Ref KeyArn
  CodeDeployDeliverystream:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: cloudwatch-logs-codedeploy
      S3DestinationConfiguration:
        BucketARN: 'arn:aws:s3:::cloudwatch-logs-firehose'
        BufferingHints:
          IntervalInSeconds: '60'
          SizeInMBs: '10'
        Prefix: 'aws/codedeploy/'
        RoleARN: !GetAtt deliveryRole.Arn
        CompressionFormat: 'UNCOMPRESSED'
        EncryptionConfiguration:
          KMSEncryptionConfig:
            AWSKMSKeyARN: !Ref KeyArn
  LambdaDeliverystream:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: cloudwatch-logs-lambda
      S3DestinationConfiguration:
        BucketARN: 'arn:aws:s3:::cloudwatch-logs-firehose'
        BufferingHints:
          IntervalInSeconds: '60'
          SizeInMBs: '10'
        Prefix: 'aws/lambda/'
        RoleARN: !GetAtt deliveryRole.Arn
        CompressionFormat: 'UNCOMPRESSED'
        EncryptionConfiguration:
          KMSEncryptionConfig:
            AWSKMSKeyARN: !Ref KeyArn
  Route53Deliverystream:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: cloudwatch-logs-route53
      S3DestinationConfiguration:
        BucketARN: 'arn:aws:s3:::cloudwatch-logs-firehose'
        BufferingHints:
          IntervalInSeconds: '60'
          SizeInMBs: '10'
        Prefix: 'aws/route53/'
        RoleARN: !GetAtt deliveryRole.Arn
        CompressionFormat: 'UNCOMPRESSED'
        EncryptionConfiguration:
          KMSEncryptionConfig:
            AWSKMSKeyARN: !Ref KeyArn
  CloudwatchRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - 'logs.us-east-1.amazonaws.com'
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            - firehose:ListDeliveryStreams
            - firehose:PutRecord
            - firehose:PutRecordBatch
            Resource:
              - !GetAtt RDSDeliverystream.Arn
              - !GetAtt CodeDeployDeliverystream.Arn
              - !GetAtt LambdaDeliverystream.Arn
              - !GetAtt Route53Deliverystream.Arn
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      ManagedPolicyArns:
      - "arn:aws:iam::aws:policy/CloudWatchLogsFullAccess"
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            # Below is needed for the lambda to log successfully to cloudwatch
            - logs:PutLogEvents
            - logs:CreateLogGroup
            - logs:CreateLogStream
            Resource: arn:aws:logs:*:*:*
          - Effect: Allow
            Action:
            - iam:PassRole
            Resource: !GetAtt CloudwatchRole.Arn
  SubscribeLogStreamsToFirehoseLambda:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          import os
          import json
          import boto3

          #Setup client
          client = boto3.client('logs')
          def handler(event, context):
            role_arn = event["filterRoleArn"]
            log_groups_total_count = 0
            log_group_successs_count = 0
            for entry in event["logs"]:
                log_prefix = entry["logGroupPrefix"]
                filter_arn = entry["subscriptionArn"]
                log_groups_response = client.describe_log_groups(logGroupNamePrefix=log_prefix)
                log_groups_list = log_groups_response['logGroups']
                # Make sure we get all the log groups by using the 'nextToken'
                while ('nextToken' in log_groups_response):
                    log_groups_response = client.describe_log_groups(logGroupNamePrefix = log_prefix, nextToken = log_groups_response['nextToken'])
                    log_groups_list = log_groups_list + log_groups_response['logGroups']
                # For each log group check the filters and subscribe if needed.
                for log_group in log_groups_list:
                  log_groups_total_count = log_groups_total_count + 1
                  filters = client.describe_subscription_filters(logGroupName=log_group['logGroupName'])
                  contains_filter = False
                  for filter in filters['subscriptionFilters']:
                      if filter['destinationArn'] == filter_arn:
                          contains_filter = True
                          break
                  if not contains_filter :
                      response = client.put_subscription_filter(
                          logGroupName=log_group['logGroupName'],
                          filterName='logmon_firehose',
                          filterPattern='',
                          destinationArn=filter_arn,
                          roleArn=role_arn)
                      if response['ResponseMetadata']['HTTPStatusCode'] == 200 :
                          print ('Successfully created a subscription filter for ' + log_group['logGroupName'])
                      else:
                          print ('Failed to create a subscription filter for ' + log_group['logGroupName'] + 'Response: ' + response)
                  log_group_successs_count = log_group_successs_count + 1
            print ('Listed ' + str(log_groups_total_count) + ' log groups. Processed: ' + str(log_group_successs_count))
      Runtime: "python2.7"
      Timeout: 300
  ScheduledRule:
    Type: AWS::Events::Rule
    Properties:
      Description: "ScheduledRule"
      ScheduleExpression: "rate(5 minutes)"
      State: "ENABLED"
      Targets:
        - Arn:
            Fn::GetAtt:
              - "SubscribeLogStreamsToFirehoseLambda"
              - "Arn"
          Id: "TargetFunctionV1"
          Input:
            !Join
              - ''
              - - '{"logs":[{"logGroupPrefix":"/aws/rds/","subscriptionArn":"'
                - !GetAtt RDSDeliverystream.Arn
                - '"},{"logGroupPrefix":"/aws/codedeploy/","subscriptionArn":"'
                - !GetAtt CodeDeployDeliverystream.Arn
                - '"},{"logGroupPrefix":"/aws/route53/","subscriptionArn":"'
                - !GetAtt Route53Deliverystream.Arn
                - '"},{"logGroupPrefix":"/aws/lambda/","subscriptionArn":"'
                - !GetAtt LambdaDeliverystream.Arn
                - '"}], "filterRoleArn":"'
                - !GetAtt CloudwatchRole.Arn
                - '"}'
  PermissionForEventsToInvokeLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref "SubscribeLogStreamsToFirehoseLambda"
      Action: "lambda:InvokeFunction"
      Principal: "events.amazonaws.com"
      SourceArn:
        Fn::GetAtt:
          - "ScheduledRule"
          - "Arn"
