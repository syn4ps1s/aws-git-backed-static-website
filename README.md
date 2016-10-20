
# Git-backed static website powered entirely by AWS

## Create CloudFormation stack for static website

    domain=example.com
    email=yourrealemail@anotherdomain.com

    template=git-backed-static-website-cloudformation.yml
    stackname=${domain/./-}-$(date +%Y%m%d-%H%M%S)
    region=us-east-1

    aws cloudformation create-stack \
      --region "$region" \
      --stack-name "$stackname" \
      --capabilities CAPABILITY_IAM \
      --capabilities CAPABILITY_NAMED_IAM \
      --template-body "file://$template" \
      --parameters \
        "ParameterKey=DomainName,ParameterValue=$domain" \
        "ParameterKey=OperatorEmail,ParameterValue=$email" \
      --tags "Key=Name,Value=$stackname"
    echo region=$region stackname=$stackname

Go to your email and approve both

 - ACM Certificate
 - SNS subscription

### Get the name servers for updating in the registrar

    hosted_zone_id=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`HostedZoneId`].[OutputValue]')
    echo hosted_zone_id=$hosted_zone_id

    aws route53 get-hosted-zone \
      --id "$hosted_zone_id"    \
      --output text             \
      --query 'DelegationSet.NameServers'

Set nameservers in your domain registrar to the above.

### Get the Git clone URL

    git_clone_url_http=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`GitCloneUrlHttp`].[OutputValue]')
    echo git_clone_url_http=$git_clone_url_http

### Use Git

    repository=$domain
    profile=$AWS_DEFAULT_PROFILE   # The correct aws-cli profile name

    git clone \
      --config 'credential.helper=!aws codecommit --profile '$profile' --region '$region' credential-helper $@' \
      --config 'credential.UseHttpPath=true' \
      $git_clone_url_http

    cd $repository

## Clean up after testing

### Delete a stack (mostly)

    aws cloudformation delete-stack \
      --region "$region" \
      --stack-name "$stackname"

Leaves behind buckets and Git repo.

### Clean up Git repo and buckets

    # WARNING! DESTROYS CONTENT AND LOGS!

    domain=...

    aws codecommit delete-repository \
      --region "$region" \
      --repository-name "$domain"

    aws s3 rm --recursive s3://logs.$domain
    aws s3 rb s3://logs.$domain
    aws s3 rm --recursive s3://$domain
    aws s3 rb s3://$domain
    aws s3 rb s3://www.$domain
    aws s3 rm --recursive s3://codepipeline.$domain
    aws s3 rb s3://codepipeline.$domain

That last command will fail, so go use the web console to delete the
versioned codepipeline bucket.

### Delete CodeCommit Git repository

    # WARNING! DESTROYS CONTENT!

    #aws codecommit delete-repository \
      --region "$region" \
      --repository-name "$repository"
