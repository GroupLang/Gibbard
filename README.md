# GroupLang-non-aws
LLM-powered telegram bot that can interact with groups of humans for varied collaborative activities. The current example use case is an agent that can rely on humans as experts in addition to using documents when answering user queries. 

The running version of the bot is currently deployed in LocalStack Lambda and is built using [LangChain](https://python.langchain.com/en/latest/index.html) as an LLM-framework. But we are [migrating](#migration)

## Features
- [x] Index and answer questions using .txt and .pdf documents, and urls from github repositories.
- [x] Ask domain experts for information.
- [x] Ask original user for clarification on a question.
- [x] Stream Chain of Thought and sends feedback requests to a dedicated moderator channel (user only sees relevant info).
- [x] Moderators can confirm or correct agent's final answer giving it feedback before sending it to the user.
- [x] Asks new group members of the moderator group for their expertise and adds user as expert when receiving the user's self-description.
- [x] Moderators can toggle feedback mode to increase bots autonomy.
- [x] Moderators can toggle debug mode to hide thoghts and logs.

### Setting up Group, Uploading Docs & Adding Experts
https://github.com/Sam1320/GroupLang/assets/33493647/9d4bce51-6a4c-459e-9759-349430b84906

### Asking Domain Experts for Help
https://github.com/Sam1320/GroupLang/assets/33493647/acbcf7f8-c048-43f1-b202-a9231ccd5c2f

### Asking Questions Regarding docs, Toggling Feedback/Debug Modes, Adding Tools
https://github.com/Sam1320/GroupLang/assets/33493647/fc02e062-b07a-4f04-b1c8-74035ef60fd3


# Getting Started
## Setup 
1. Create an [OpenAI account](https://openai.com/api/) and [get an API Key](https://platform.openai.com/account/api-keys).
2. Setup your Telegram bot. You can follow [this instructions](https://core.telegram.org/bots/tutorial#obtain-your-bot-token) to get your token.
3. Create a Pinecone account and get an API key.
4. Create a Serper account and get an API key.

## Installation
1. Install Python using [pyenv](https://github.com/pyenv/pyenv-installer) or your prefered Python installation.
2. Create a virtual environment: `python3 -m venv .venv`.
3. Activate you virtual environment: `source .venv/bin/activate`.
3. Install dependencies: `pip install -r requirements.txt`.
4. [Install the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
5. Install Docker in your system
6. Install LocalStack using `pip install localstack`
7. start LocalStack in a terminal/cmd window using `localstack start` (make sure docker is running in your system (check with `sudo docker images -a`))
8. makesure localstack is running on port 4566
9. If you are on Mac/ Linux, Open Terminal. If you are using Windows, open PowerShell and execute the below command
10. ```
   aws configure
   > AWS Access Key ID [None]: test
   > AWS Secret Access Key [None]: test
   > Default region name [None]: us-east-1
   > Default output format [None]:
   ```
11. Create new private bucket using command `aws s3api create-bucket --bucket <private-bucket-name> --endpoint-url='http://localhost:4566'`
12. Create new public bucket using command `aws s3api create-bucket --bucket <public-bucket-name> --endpoint-url='http://localhost:4566'`
13. two buckets - one public used as storage for new bots created and one private used for storing global defaults.
14. confirm created buckets using command `aws s3 ls --endpoint-url='http://localhost:4566'`
15. Edit `.chalice/config.json` & `chalicelib/credentials.py` and stablish the configurations:
- `OPENAI_API_KEY` with the value of your Open AI API Token.
- `S3_BUCKET`: with the bucket name you created previously.
- `S3_PRIVATE_BUCKET`: with the other bucket name,
- `SERPER_API_KEY`: with your Serper API key,
- `PINECONE_API_KEY`: with your Pinecone API key,
- `PINECONE_ENVIRONMENT` with your Pinecone environment.
- `MAIN_BOT_NAME` with the *username* of your main bot. (without @)
- `MAIN_MOD_USERNAME` with the telegram username of the user you want to be the main moderator of the bot. (without @) (you can find your telegram username using @userinfobot in telegram)
- `MAIN_MOD_ID` with the telegram id of the user you want to be the main moderator of the bot. (you can find your telegram id using @userinfobot in telegram)

## Deployment
1. store defaults in the s3 private bucket by running `python scripts/register_defaults_in_s3.py`.
2. install local chalice using command `pip install chalice-local`
3. Run `chalice-local deploy`. (takes time)
4. Run `aws lambda create-function-url-config --function-name grouplang-dev-message-handler --auth-type NONE --endpoint-url='http://localhost:4566'`
5. Copy the created function URL.
6. Run ` aws lambda publish-layer-version --layer-name git-layer --compatible-runtimes python3.8 --zip-file fileb://git-layer.zip --endpoint-url=http://localhost:4566 `
7. Run `ngrok http <host:port>` (get host and port from created function url)
  Example :
  if function url is : http://h25w1l1o9k798d062saknvd1693tkjjk.lambda-url.us-east-1.localhost.localstack.cloud:4566/
  ```
  ngrok http h25w1l1o9k798d062saknvd1693tkjjk.lambda-url.us-east-1.localhost.localstack.cloud:4566
  ```
8. copy new function url provided by ngrok (https version)
6. run `python scripts/set_bot_webhook.py <YOUR_FUNCTION_URL> <YOUR_TELEGRAM_TOKEN>` to stablish your Telegram webhook to point to you AWS Lambda. (telegram token generated by botfather)
7. run `python scripts/register_bot.py <YOUR_TELEGRAM_TOKEN>` to register your main bot in the private S3 bucket. (telegram token generated by botfather)
8. run `python scripts/register_lambda.py <YOUR_FUNCTION_NAME> <YOUR_FUNCTION_URL>` to register your lambda function in s3 (will be used to set the webhooks of community bots programatically.

Now you can go an setup your bot & group in telegram!.

# Migration
We are currently migrating the hosting to [Modal](https://modal.com/) and replacing LangChain with [Guidance](https://github.com/microsoft/guidance). The main reasons are the following:

1. Although LangChain is nice to get started quickly, it is not [well suited](https://github.com/hwchase17/langchain/issues/1364#issuecomment-1560895510) for a serverless framework. Also due to its extremely nested design it is not easy to keep an intuitive overview of the prompts and parsers used by each agent. [Guidance](https://github.com/microsoft/guidance) solves this by representing programs as simple strings which allows to see all prompts at once.
2. Modal is built explicitly for ML applications in mind, is very easy to setup and manage compared to AWS Lambda, and offersthe same instant feedback loop you have when you develop locally. 
3. [Guidance](https://github.com/microsoft/guidance)'s [token healing](https://github.com/microsoft/guidance/blob/main/notebooks/art_of_prompt_design/prompt_boundaries_and_token_healing.ipynb) automatically deals with the uninteded consequences of greedy tokenizations used by most language models. 

The migration process is taking place in the [switch_to_guidance](https://github.com/Sam1320/GroupLang/tree/switch_to_guidance) branch.

# Next Steps
- [x] Handle urls of repositories to load information in the knowledge base.
- [ ] Create summary of documents before indexing and add list of summaries to the query agent prompt so it knows what it knows.
- [ ] Use single config.json (or other format) to store the community parameters
- [ ] Use one namespace per community when indexing in vector store.
- [ ] Index (bad answer, feedback) tuples to improve quality of answers overtime.
- [ ] Use google api for search directly instead of serper api
- [ ] Add timeout + exponential backoff when waiting for expert/moderator feedback.
- [ ] Finish Guidance/Modal migration
There is also a lot of additional functionality *already implemented* but currently not integrated. The corresponging code is currently commented out.
### WIP:
- [ ] Creating tabular data from collections of free-form user messages.
- [ ] Matching users based of self description
- [ ] Create & summarize user description based on messages.
- [ ] Use Plan & Execute framework to solve more complicated tasks.


## Other Notes & Information

## License
GroupLang is licensed under the MIT License. For more information, see the [LICENSE](LICENSE) file.
