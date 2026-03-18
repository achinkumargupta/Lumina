# Lumina
Running Lumina Locally — Complete Setup Guide
Prerequisites
Install these first if you don't have them:

Node.js 20+

# Check version
node --version  # should be v20 or higher
# Install via nvm if needed
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
nvm use 20
pnpm (this project requires pnpm, not npm or yarn)

npm install -g pnpm
pnpm --version  # confirm it's installed
PostgreSQL 16

# macOS (via Homebrew)
brew install postgresql@16
brew services start postgresql@16
# Ubuntu/Debian
sudo apt install postgresql-16
sudo systemctl start postgresql
Step 1 — Get the Code
Extract the downloaded archive:

tar -xzf lumina-knowledge-base.tar.gz
cd lumina-knowledge-base
Step 2 — Create the Database
# macOS
psql postgres -c "CREATE DATABASE lumina;"
# Ubuntu (switch to postgres user first)
sudo -u postgres psql -c "CREATE DATABASE lumina;"
Then get the connection string. It will look like:

postgresql://your_username@localhost:5432/lumina
On macOS with Homebrew, your username is usually your Mac login name. On Ubuntu it's postgres. Test it:

psql postgresql://your_username@localhost:5432/lumina -c "SELECT 1;"
Step 3 — Set Environment Variables
Create a .env file inside the lumina-knowledge-base folder:

cat > .env << 'EOF'
DATABASE_URL=postgresql://your_username@localhost:5432/lumina
AI_INTEGRATIONS_OPENAI_API_KEY=sk-your-openai-api-key-here
AI_INTEGRATIONS_OPENAI_BASE_URL=https://api.openai.com/v1
EOF
Get your OpenAI API key from https://platform.openai.com/api-keys

Step 4 — Install Dependencies
cd lumina-knowledge-base
pnpm install
This installs all packages across all workspaces (API server, frontend, DB library, etc.) in one go.

Step 5 — Push the Database Schema
This creates all the tables in your PostgreSQL database:

# Load env vars, then push schema
export $(cat .env | xargs)
pnpm --filter @workspace/db run push
You should see Drizzle confirm it created the documents, threads, and thread_messages tables (plus their enums).

Step 6 — Start the API Server
Open Terminal 1:

cd lumina-knowledge-base
export $(cat .env | xargs)
PORT=8080 pnpm --filter @workspace/api-server run dev
You should see:

Server listening on port 8080
Test it:

curl http://localhost:8080/api/documents
# Should return: []
Step 7 — Start the Frontend
Open Terminal 2:

cd lumina-knowledge-base
export $(cat .env | xargs)
PORT=19350 BASE_PATH=/ pnpm --filter @workspace/knowledge-base run dev
You should see:

VITE v7.x.x  ready in ...ms
➜  Local:   http://localhost:19350/
The Vite dev server already has a built-in proxy that forwards all /api requests to http://localhost:8080, so both services communicate automatically.

Step 8 — Open the App
Visit http://localhost:19350 in your browser.

You should see the Lumina welcome screen with an empty library. Click New Document to create your first document.

Quick Reference — All Commands Together
# Terminal 0 — one-time setup
tar -xzf lumina-knowledge-base.tar.gz
cd lumina-knowledge-base
pnpm install
export $(cat .env | xargs)
pnpm --filter @workspace/db run push
# Terminal 1 — API server (keep running)
export $(cat .env | xargs) && PORT=8080 pnpm --filter @workspace/api-server run dev
# Terminal 2 — Frontend (keep running)
export $(cat .env | xargs) && PORT=19350 BASE_PATH=/ pnpm --filter @workspace/knowledge-base run dev
Common Issues
Problem	Fix
pnpm: command not found	Run npm install -g pnpm
DATABASE_URL must be set	Make sure you ran `export $(cat .env
AI_INTEGRATIONS_OPENAI_BASE_URL must be set	Same — env vars must be exported in Terminal 1
ECONNREFUSED on API calls	API server isn't running — check Terminal 1
relation "documents" does not exist	Schema wasn't pushed — repeat Step 5
Port already in use	Change PORT=8080 or PORT=19350 to any free port (update both terminals to match)
Error: Use pnpm instead	You ran npm install — use pnpm install instead
