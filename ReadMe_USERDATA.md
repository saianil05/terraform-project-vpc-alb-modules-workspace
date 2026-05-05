🧱 🏗️ Your Architecture (What you built)
🌐 High-Level Design

Internet
   ↓
External ALB
   ↓
Web Tier (Public Subnets)
   ↓
Internal ALB
   ↓
App Tier (Private Subnets)
   ↓
DB Tier (Private Subnets - RDS)

🌐 External Load Balancer (Internet-facing ALB)

✅ Purpose:
Entry point from internet
Handles:
HTTPS (SSL termination)
Routing to web tier
Load balancing across AZs

👉 Without this:

You’d expose EC2 directly (bad security)
🔒 Internal Load Balancer (Private ALB)

✅ Purpose:
Routes traffic inside VPC only
Web → App communication

👉 Why needed?

Decouples Web & App layers
Allows scaling App independently
Supports microservices later

🌍 Why NGINX in Web Tier?

👉 This is where many people get confused.

❓ “Why not directly ALB → App?”

You can, but production systems often include NGINX because:

✅ 1. Reverse Proxy
Client → NGINX → Internal ALB → App
NGINX forwards requests internally
✅ 2. Static Content Handling
Serves HTML, CSS, JS faster than Node
✅ 3. Security Layer
Hide backend structure
Add headers, filtering
✅ 4. Flexibility
URL routing
Blue/Green deployments
Caching
🔥 Real Reason (Important)

👉 External ALB = edge routing
👉 NGINX = application-level control
👉 Internal ALB = service routing

🔍 2️⃣ Your Scripts — Line-by-Line Explanation

📄 🧾 web_user_data.sh

🟢 Shebang + safety

#!/bin/bash
set -e
#!/bin/bash → run with bash

set -e → stop if any command fails

🔄 Update + install

dnf update -y
dnf install -y nginx git

Updates OS
Installs:
NGINX (web server)
Git (to pull code)

📥 Clone repo

cd /home/ec2-user
git clone https://github.com/... || true

Downloads your project
|| true → ignore error if already cloned

📄 Copy script

cp -f .../web.sh /home/ec2-user/web.sh
chmod +x /home/ec2-user/web.sh

Copies helper script
Makes it executable

🔧 Replace placeholder (VERY IMPORTANT)

sed -i "s|REPLACE-WITH-INTERNAL-LB-DNS|__APP_ALB_DNS__|g"

👉 This updates nginx config:

proxy_pass http://internal-alb

⚠️ Must be replaced by Terraform dynamically

🔁 Apply nginx config

mv /etc/nginx/nginx.conf /etc/nginx/nginx-backup.conf || true
cp .../nginx.conf /etc/nginx/nginx.conf

Backup default config
Replace with your custom config

▶️ Run web.sh

/home/ec2-user/web.sh

Likely sets up static content / frontend

✅ Validate config

nginx -t

Checks if config is valid before starting

🔄 Start service

systemctl restart nginx
systemctl enable nginx

Restart nginx
Enable on boot
📄 🧾 app_user_data.sh

🟢 Setup

#!/bin/bash
set -e

Same as before

🔄 Install dependencies

dnf install -y git mysql jq
dnf install -y nodejs npm

git → clone repo
mysql → connect to RDS
jq → parse JSON (for secrets)
node/npm → run app

📥 Clone repo

git clone ...

Same as web

📁 Define path
APP_DIR=...

Just a variable for reuse

🔐 Environment variables
cat <<EOF > /etc/profile.d/app_env.sh
export SECRET_NAME="${secret_name}"
export REGION="${region}"
EOF

👉 This is powerful:

Stores env variables globally
Used by app to fetch secrets

🔄 Load env

source /etc/profile.d/app_env.sh

Makes variables active
📦 Copy app files

cp -rf .../app_files /home/ec2-user
cp -rf .../app.sh /home/ec2-user
chmod +x /home/ec2-user/app.sh

Copies application code
Prepares startup script

▶️ Run app

sudo /home/ec2-user/app.sh

👉 This likely:

Runs Node.js server
Starts API

📂 Move to app folder
cd /home/ec2-user/app_files

🎉 Final logs
echo "App tier setup completed"

Just logging info
🔥 Final Understanding (This is the key)
Why all this complexity?

👉 Because production systems need:

| Layer        | Responsibility  |
| ------------ | --------------- |
| External ALB | Internet entry  |
| NGINX        | Proxy + control |
| Internal ALB | Service routing |
| App          | Business logic  |
| DB           | Data            |
