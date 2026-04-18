# ShadowBlade Executable Instructions Only

All commands and prompts you need to execute, in order. Shell commands go in the terminal. Gemini prompts go inside the `gemini` CLI.

---

## Part 1 — Setup

### 1.1 Claim free credit (browser)

Open: url from GDG → sign in with personal Gmail → accept ToS.

### 1.2 Open Cloud Shell (browser)

Go to <https://console.cloud.google.com> → click terminal icon (`>_`) → click **Open Editor**.

### 1.3 Download starter code

```bash
git clone https://github.com/weimeilin79/agentverse-developer.git
chmod +x ~/agentverse-developer/*.sh
```

### 1.4 Run setup script (press Enter when prompted for Project ID)

```bash
cd ~/agentverse-developer
./init.sh
```

### 1.5 Enable services

```bash
gcloud config set project $(cat ~/project_id.txt) --quiet
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com \
  run.googleapis.com cloudbuild.googleapis.com aiplatform.googleapis.com \
  iam.googleapis.com cloudresourcemanager.googleapis.com
```

### 1.6 Update Gemini CLI

```bash
npm update -g @google/gemini-cli
```

---

## Part 2 — Gemini CLI

### 2.1 Start Gemini (choose NO when asked about editor connection)

```bash
cd ~/agentverse-developer
mkdir playground
cd playground
gemini
```

### 2.2 Inside Gemini, run each:

```
/help
/tools
!ls -l
/memory add "My name is [your name]. I'm learning about AI agents."
/memory show
```

### 2.3 First vibe-coding prompt (inside Gemini)

```
Write a Python script called hello.py that prints "I built my first AI-generated file" and the current date.
```

Then verify (inside Gemini):

```
!ls
!cat hello.py
!python3 hello.py
```

Exit Gemini: **Ctrl+C twice**.

---

## Part 3 — Website + Git

### 3.1 Start Gitea

```bash
cd ~/agentverse-developer
./gitea.sh
```

Then in browser: **Web Preview → Change port → 3005 → login `dev` / `dev`**.

### 3.2 Connect Gemini to Gitea (in terminal)

```bash
if [ ! -f ~/.gemini/settings.json ]; then
  echo '{"mcpServers":{"gitea":{"url":"http://localhost:8085/sse"}}}' > ~/.gemini/settings.json
else
  jq '. * {"mcpServers":{"gitea":{"url":"http://localhost:8085/sse"}}}' ~/.gemini/settings.json > ~/.gemini/settings.json.tmp && mv ~/.gemini/settings.json.tmp ~/.gemini/settings.json
fi
```

### 3.3 Build the website

```bash
cd ~/agentverse-developer/playground
gemini
```

Inside Gemini:

```
/mcp
```

```
Create a personal profile website in the current folder. Dark theme, electric blue accents. Two files: index.html and styles.css. Use flexbox for a two-column layout. Include a placeholder spot for a profile picture. Make the code clean and commented. Don't start any server.
```

Exit Gemini (**Ctrl+C twice**), then preview:

```bash
python -m http.server
```

Browser: **Web Preview → port 8000**. Then **Ctrl+C** to stop.

### 3.4 Push to Gitea

```bash
gemini
```

Inside Gemini:

```
Create a new Gitea repository named 'my-profile' with description 'My first AI-built website'. Don't add any content yet.
```

```
Using the Gitea tool, push index.html and styles.css to the 'my-profile' repository.
```

### 3.5 File and close an issue (inside Gemini)

```
File an issue in the my-profile repo titled "Profile image is missing". Use the Gitea tool and the 'dev' user account.
```

```
Close issue #1 in the my-profile repo. Use the 'dev' user account.
```

---

## Part 4 — Build the Agent

### 4.1 Set up workspace

```bash
cd ~/agentverse-developer/shadowblade
. ~/agentverse-developer/set_env.sh
```

### 4.2 Write coding rules

```bash
cat << 'EOF' > GEMINI.md
### Coding Rules for This Project
- Use Python 3 with type hints on every function.
- Every function needs a docstring explaining what it does.
- Use snake_case for variables and functions, PascalCase for classes.
- Keep code clean and readable.
EOF
```

### 4.3 Copy prebuilt code

```bash
cp ~/agentverse-developer/working_code/agent.py ~/agentverse-developer/shadowblade/
cp ~/agentverse-developer/working_code/mcp_server.py ~/agentverse-developer/shadowblade/
```

### 4.4 Install and run

```bash
cd ~/agentverse-developer
python -m venv env
source env/bin/activate
pip install --upgrade pip
pip install -r shadowblade/requirements.txt
adk run shadowblade
```

Inside the agent prompt:

```
We're stuck against 'Perfectionism'. Its weakness is 'Elegant Sufficiency'. Break us out!
```

```
'Dogma' blocks our path. Its weakness is 'Revolutionary Rewrite'. Take it down.
```

Exit: **Ctrl+C twice**.

### 4.5 Run tests

```bash
cp ~/agentverse-developer/working_code/test_agent_initiative.py ~/agentverse-developer/shadowblade/
source ~/agentverse-developer/env/bin/activate
cd ~/agentverse-developer
. ~/agentverse-developer/set_env.sh
pytest test_agent_initiative.py
```

---

## Part 5 — Deploy + Clean Up

### 5.1 Deploy to Cloud Run

```bash
. ~/agentverse-developer/set_env.sh
gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION \
  --description="Agent repo" 2>/dev/null || echo "Already exists, moving on"
```

```bash
for ROLE in artifactregistry.admin cloudbuild.builds.editor run.admin \
  iam.serviceAccountUser aiplatform.user logging.logWriter logging.viewer; do
    gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member="serviceAccount:$SERVICE_ACCOUNT_NAME" \
      --role="roles/$ROLE" --quiet
done
```

```bash
sed -i 's|COPY ./shadowblade|COPY .|g' ~/agentverse-developer/shadowblade/Dockerfile
sed -i 's|COPY shadowblade|COPY .|g' ~/agentverse-developer/shadowblade/Dockerfile

cd ~/agentverse-developer
gcloud builds submit ./shadowblade \
  --tag ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/my-agent:latest
```

```bash
gcloud run deploy my-agent \
  --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/my-agent:latest \
  --region=${REGION} \
  --allow-unauthenticated \
  --set-env-vars="A2A_HOST=0.0.0.0,A2A_PORT=8080,GOOGLE_GENAI_USE_VERTEXAI=TRUE" \
  --min-instances=1 \
  --project=${PROJECT_ID}
```

Open this URL in your browser (append /.well-known/agent-card.json to your service URL):
```bash
https://my-agent-xxxxx-uc.a.run.app/.well-known/agent-card.json
```

### 5.2 Clean up

```bash
. ~/agentverse-developer/set_env.sh
gcloud run services delete my-agent --region=${REGION} --quiet
gcloud artifacts repositories delete ${REPO_NAME} --location=${REGION} --quiet
rm -rf ~/agentverse-developer ~/.gemini
rm -f ~/project_id.txt
```
