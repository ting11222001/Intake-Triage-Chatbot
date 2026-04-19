# Notes

## Scripts

`main.py`: terminal version, to test quickly locally

`api.py`: FastAPI version, the one that gets deployed

To run the terminal version:
```
python main.py
```

To run the API version:
```
uvicorn api:app --reload
```

Then, go to this API doc:
```
http://localhost:8000/docs
```





## Test Cases and Results for `main.py`

### Test 1: Billing, high confidence

**Input**
```
Name     → Priya Sharma
Problem  → Our client received an invoice for Phase 2 but we have not signed off on that milestone yet
Urgent?  → high
```

**Expected:** routes to billing, confidence above 70

**Output**
```
Bot: You will be routed to our billing team.
```



### Test 2: Account access

**Input**
```
Name     → David Chen
Problem  → Our client cannot log into the EY client portal to view their deliverables
Urgent?  → medium
```

**Expected:** routes to account access

**Output**
```
Bot: You will be routed to our account access team.
```



### Test 3: Vague problem, human fallback

**Input**
```
Name     → Tom Walsh
Problem  → The partner wants something done about the report but I am not sure what exactly
Urgent?  → low
```

**Expected:** confidence below 70, human agent message appears

**Output**
```
Bot: I'm not confident enough to classify route you automatically.
Bot: A human agent will review your case and contact you shortly.
```



### Test 4: Technical issue

**Input**
```
Name     → Maria Costa
Problem  → The PowerPoint live link stopped working during our client workshop this morning
Urgent?  → high
```

**Expected:** routes to technical support

**Output**
```
Bot: You will be routed to our technical support team.
```



### Test 5: General enquiry

**Input**
```
Name     → James Okafor
Problem  → We want to understand what AI tools EY can offer for our supply chain project
Urgent?  → medium
```

**Expected:** routes to general enquiry

**Output**
```
Bot: You will be routed to our general enquiry team.
```





## Debug Note

Early test run showed the raw parsed response before routing logic was confirmed:

```
CATEGORY: billing
CONFIDENCE: 95
REASON: The client explicitly states they were charged twice on their credit card, which is a clear billing issue.

lines: ['CATEGORY: billing', 'CONFIDENCE: 95', 'REASON: The client explicitly states they were charged twice on their credit card, which is a clear billing issue.']
```

The `lines` print statement was used for debugging and can be removed.

The response from:
```python
response = client.messages.create(
    model="claude-haiku-4-5",       # simple classification, so we can use a smaller model
    max_tokens=200,                  # stop after 200 tokens at most
    messages=[{"role": "user", "content": prompt}]
)
```

looked like this:
```
Message(id='msg_01WSBbX73t5KqtyhCjcdvM2e', container=None, content=[TextBlock(citations=None, text='CATEGORY: account access\nCONFIDENCE: 95\nREASON: The client explicitly states they cannot log in to their account, which is a clear account access issue.', type='text')], model='claude-haiku-4-5-20251001', role='assistant', stop_details=None, stop_reason='end_turn', stop_sequence=None, type='message', usage=Usage(cache_creation=CacheCreation(ephemeral_1h_input_tokens=0, ephemeral_5m_input_tokens=0), cache_creation_input_tokens=0, cache_read_input_tokens=0, inference_geo='not_available', input_tokens=181, output_tokens=38, server_tool_use=None, service_tier='standard'))
```

And `result = response.content[0].text` looked like this:
```
Bot (internal): CATEGORY: account access
CONFIDENCE: 95
REASON: The client explicitly states they cannot log into their account, which is a clear account access issue.
```





## Billing

Anthropic API needs a separate credit balance. Once topped up, create a new API key to make sure the API key and the billing balance are in sync.

Check the credit balance here: https://platform.claude.com/settings/billing





## Test Cases and Results for `api.py`

For the `/classify` endpoint in the docs UI, put these in the request body.

To run:
```
uvicorn api:app --reload
```

FastAPI gives you a free test page here: http://127.0.0.1:8000/docs

See the two endpoints, `GET /questions` and `POST /classify`. Click on `/classify`, then "Try it out", and paste the input as the request body.



### Test 1: Billing, high confidence

**Input**
```json
{
  "answers": ["Priya Sharma", "Our client received an invoice for Phase 2 but we have not signed off on that milestone yet", "high"]
}
```

**Result**
```json
{
  "category": "billing",
  "confidence": 95,
  "reason": "The client is disputing an invoice for a milestone that hasn't been completed or approved, which is clearly a billing issue.",
  "routed_to": "billing team"
}
```



### Test 2: Account access

**Input**
```json
{
  "answers": ["David Chen", "Our client cannot log into the EY client portal to view their deliverables", "medium"]
}
```

**Result**
```json
{
  "category": "account access",
  "confidence": 95,
  "reason": "The client explicitly states they cannot log into the EY client portal, which is a clear account access issue.",
  "routed_to": "account access team"
}
```



### Test 3: Vague problem, human fallback

**Input**
```json
{
  "answers": ["Tom Walsh", "The partner wants something done about the report but I am not sure what exactly", "low"]
}
```

**Result**
```json
{
  "category": "general enquiry",
  "confidence": 45,
  "reason": "The client is unclear about the specific problem, making it difficult to classify, but it appears to be a general enquiry rather than a technical, billing, or account access issue.",
  "routed_to": "human review"
}
```



### Test 4: Technical issue

**Input**
```json
{
  "answers": ["Maria Costa", "The PowerPoint live link stopped working during our client workshop this morning", "high"]
}
```

**Result**
```json
{
  "category": "technical support",
  "confidence": 95,
  "reason": "Maria is reporting a non-functional PowerPoint live link feature that failed during a client meeting, which is clearly a technical issue requiring immediate resolution.",
  "routed_to": "technical support team"
}
```



### Test 5: General enquiry

**Input**
```json
{
  "answers": ["James Okafor", "We want to understand what AI tools EY can offer for our supply chain project", "medium"]
}
```

**Result**
```json
{
  "category": "general enquiry",
  "confidence": 92,
  "reason": "The client is asking for information about AI tools and services available for a supply chain project, which is a general product inquiry rather than a specific technical, billing, or account issue.",
  "routed_to": "general enquiry team"
}
```



### Quick Test

Paste this into the `/classify` request body:
```json
{
  "answers": ["Li Ting", "I cannot log into my account", "high"]
}
```

Terminal will print:
```
Received answers:  ['Li Ting', 'I cannot log into my account', 'high']

Result:  ['CATEGORY: account access', 'CONFIDENCE: 95', 'REASON: The client explicitly states they cannot log into their account, which is a clear account access issue.']
```

Response:
```json
{
  "category": "account access",
  "confidence": 95,
  "reason": "The client explicitly states they cannot log into their account, which is a clear account access issue.",
  "routed_to": "account access team"
}
```

## Setup Streamlit

Streamlit turns a Python script into a web page. No HTML or CSS needed.

Install Streamlit first:
```
pip install streamlit
```

Create a new file called `app.py`.

Terminal 1 (runs the API):
```
uvicorn api:app --reload
```

Terminal 2 (runs the frontend):
```
streamlit run app.py
```

Streamlit will open your browser automatically at http://localhost:8501.


## Deployment

URLs are defined in `config.py`.

To switch, just change ENV in your .env file:
```
ENV=local
```
or
```
ENV=aws
```

Double check:
```
aws --version
sam --version
aws configure list
```

In `requirements.txt`:
```
anthropic
fastapi
mangum        # mangum a small library that makes FastAPI work inside Lambda. Lambda
python-dotenv
```

In `api.py`:
```
from mangum import Mangum
handler = Mangum(app)
```

Also, install:
```
pip install mangum
```

When you run `uvicorn api:app` locally, `uvicorn` is the thing that receives web requests and passes them to your FastAPI app.
But on Lambda, there is no `uvicorn`. Lambda receives a request in its own format and looks for a function called handler to call.
`Mangum` sits in between. It receives the Lambda-format request, translates it into a format FastAPI understands, and passes it through.
So:

- Locally: `uvicorn` talks to app
- On Lambda: handler receives the request, `Mangum` translates it, then passes it to app

The `template.yaml` line Handler: `api.handler` tells Lambda exactly where to find it, meaning look in `api.py` for a thing called `handler`.


### AWS CLI

Store my API key in AWS SSM:
```
aws ssm put-parameter --name "/intake-chatbot/anthropic-api-key" --value "sk-ant-api03-xxxxxxxxxxxx" --type SecureString
```

Now build and deploy:
```
cd <project-root-path>
sam build
sam deploy --guided
```

For the guided prompts, use these answers:
```
Stack name: intake-chatbot
Region: ap-southeast-2
Confirm changes: Y
Allow IAM role creation: Y
Save arguments to config file: Y
```

At the end you will see a URL that looks like:
https://abc123.execute-api.ap-southeast-2.amazonaws.com

Copy that URL and paste it into `config.py` under "aws".

### PowerShell error if I run sam build directly

In powershell, before `sam build`, run these first.

First command:
```
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
```

Windows blocks PowerShell scripts by default for security. This command temporarily allows scripts to run, but only for the current PowerShell window. It does not change your system settings permanently.

Second command:
```
.\venv\Scripts\Activate.ps1
```
This activates your Python virtual environment. After this runs, your terminal uses the Python and packages inside the venv folder, not your system Python.

Then run `sam build` shows this:
```
Building codeuri: C:\Users\Li-Ting\Documents\Projects\intake-chatbot runtime: python3.12 architecture: x86_64 functions:
IntakeChatbotFunction                                                                                                   
 Running PythonPipBuilder:ResolveDependencies                                                                           
 Running PythonPipBuilder:CopySource                                                                                    

Build Succeeded

Built Artifacts  : .aws-sam\build
Built Template   : .aws-sam\build\template.yaml

Commands you can use next
=========================
[*] Validate SAM template: sam validate
[*] Invoke Function: sam local invoke
[*] Test Function in the Cloud: sam sync --stack-name {{stack-name}} --watch
[*] Deploy: sam deploy --guided
```

Then run `sam deploy --guided`.

Here are my answers to the prompt:
```
  Stack Name [sam-app]: intake-chatbot
  AWS Region [ap-southeast-2]: ap-southeast-2
  Confirm changes before deploy [y/N]: y
  Allow SAM CLI IAM role creation [Y/n]: y
  Disable rollback [y/N]: n
  IntakeChatbotFunction has no authentication. Is this okay? [y/N]: y
  Save arguments to configuration file [Y/n]:       # the rest I just pressed Enter to accept the defaults
  SAM configuration file [samconfig.toml]: 
  SAM configuration environment [default]: 
```

At the end I should see a URL that looks like:
```
https://abc123.execute-api.ap-southeast-2.amazonaws.com
```

Copy that URL and paste it into `config.py` under "aws".


### Check the shape of `/intake-chatbot/anthropic-api-key` parameter in AWS SSM 

To check the API key I stored under the name `/intake-chatbot/anthropic-api-key` in AWS SSM, run this:
```
aws ssm get-parameter --name "/intake-chatbot/anthropic-api-key" --with-decryption
```

`--with-decryption`: decrypt the value before returning it as the key was stored as SecureString

It will print something like:
```
{
    "Parameter": {
        "Name": "/intake-chatbot/anthropic-api-key",
        "Type": "SecureString",
        "Value": "sk-ant-api03-xxxx",
        "Version": 1,
        ...
    }
}
```

which is what `response` from `ssm.get_parameter()` in `api.py` would look like.

### Get the API Gateway URL after deploy

Once it shows:
```
Successfully created/updated stack - intake-chatbot in ap-southeast-2
```

Do this to find the public URL of API Gateway:
```
sam list endpoints --stack-name intake-chatbot --region ap-southeast-2
```

Which is this:
```
https://<id>.ap-southeast-2.amazonaws.com/$default
```

Save it in the `.env` file as `AWS_LAMBDA_URL`.

Run this to confirm Lambda is working:
```
curl https://<id>.ap-southeast-2.amazonaws.com/questions
```

It will print:
```
Security Warning: Script Execution Risk                                                                                                                                      
Invoke-WebRequest parses the content of the web page. Script code in the web page might be run when the page is parsed.
      RECOMMENDED ACTION:
      Use the -UseBasicParsing switch to avoid script code execution.

      Do you want to continue?
    
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): y


StatusCode        : 200
StatusDescription : OK
Content           : {"questions":["What is your name?","Can you describe the problem you are having?","How urgent is this for you? (low / medium / high)"]}
RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    Apigw-Requestid: cESC8gznSwMEMqA=
                    Content-Length: 135
                    Content-Type: application/json
                    Date: Sun, 19 Apr 2026 12:26:59 GMT
                    
                    {"questions":["What is your name...
Forms             : {}
Headers           : {[Connection, keep-alive], [Apigw-Requestid, cESC8gznSwMEMqA=], [Content-Length, 135], [Content-Type, application/json]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : System.__ComObject
RawContentLength  : 135
```

### Streamlit

Streamlit Community Cloud (free, easiest)
- Push code to GitHub
- Go to https://share.streamlit.io
- Connect repo and deploy `app.py`
- Get a free public URL like https://yourname-intake-chatbot.streamlit.app

Settings > Secrets:
```
ANTHROPIC_API_KEY="..."
AWS_LAMBDA_URL="https://<id>.ap-southeast-2.amazonaws.com"
```

ENV not set in the streamlit dashboard to let it route to aws lambda in `config.py`.