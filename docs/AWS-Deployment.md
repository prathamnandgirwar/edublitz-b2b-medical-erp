# MedERP — AWS EC2 Deployment Guide

---

## What You Will Deploy

- 3 Java backend services (user, product, order)
- React frontend served by Nginx
- MongoDB Atlas as the database (free cloud hosted — no install needed)

---

## Table of Contents

1. [Create MongoDB Atlas](#1-create-mongodb-atlas)
2. [Launch EC2 Instance](#2-launch-ec2-instance)
3. [Connect to EC2](#3-connect-to-ec2)
4. [Install Everything](#4-install-everything)
5. [Clone the Repo](#5-clone-the-repo)
6. [Configure Environment Files](#6-configure-environment-files)
7. [Build the Backend](#7-build-the-backend)
8. [Run Backend as Services](#8-run-backend-as-services)
9. [Build and Deploy Frontend](#9-build-and-deploy-frontend)
10. [Configure Nginx](#10-configure-nginx)
11. [Bootstrap Users and Orgs](#11-bootstrap-users-and-orgs)
12. [Test the Application](#12-test-the-application)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Create MongoDB Atlas

MongoDB Atlas is a free cloud database. You do NOT need to install MongoDB on your server.

1. Go to https://cloud.mongodb.com and create a free account
2. Click **Create a new Project** → name it `medical-erp` → **Create Project**
3. Click **Build a Database** → choose **M0 Free** → choose a region close to your EC2 → **Create**
4. When prompted, create a **Database User**:
   - Username: `admin`
   - Click **Autogenerate Secure Password**
   - **COPY AND SAVE THIS PASSWORD** — you cannot see it again
   - Click **Create Database User**
5. Under **Network Access** → **Add IP Address** → **Allow Access from Anywhere** (`0.0.0.0/0`) → **Confirm**
6. Go to **Database** → **Connect** → **Drivers** → copy the connection string

The connection string looks like this:
```
mongodb+srv://admin:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority&appName=Cluster0
```

Save this string. You will use it in Step 6.

---

## 2. Launch EC2 Instance

1. Go to AWS Console → EC2 → **Launch Instance**
2. Settings:
   - **Name**: `medical-erp`
   - **OS**: Ubuntu Server 22.04 LTS
   - **Instance type**: `t3.medium` (minimum — needs at least 4GB RAM for 3 Java services)
   - **Key pair**: Create new → download the `.pem` file → keep it safe
   - **Storage**: 20GB
3. Under **Network settings** → **Edit**:
   - Allow SSH (port 22) — set to **My IP** only
   - Allow HTTP (port 80) — from anywhere
   - **Do NOT open ports 8081, 8082, 8083** — these are proxied internally by Nginx
4. Click **Launch Instance**
5. Once running, copy the **Public IPv4 address** — you need this throughout

---

## 3. Connect to EC2

On your local machine, in the same folder as your `.pem` file:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<your-ec2-public-ip>
```

> If you see "Permission denied", check you used `ubuntu@` not `ec2-user@` for Ubuntu AMIs.

---

## 4. Install Everything

Run these commands one by one on your EC2 instance:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java 17, Maven, Git, Nginx, Curl
sudo apt install -y openjdk-17-jdk maven git nginx curl

# Verify
java -version    # must show 17.x
mvn -version
nginx -v
```

Install Node.js via nvm:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 18
node -v    # must show v18.x
npm -v
```

Add swap space (prevents out-of-memory crashes with 3 Java services running):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h    # confirm Swap row shows 2.0Gi
```

---

## 5. Clone the Repo

```bash
cd ~
git clone https://github.com/Rohit-1920/medical_erp.git
cd medical_erp
ls
```

You should see: `frontend  user-service  product-service  order-service  seed.sh  README.md`

---

## 6. Configure Environment Files

### Generate a JWT secret (do this first)

```bash
openssl rand -hex 32
```

Copy the output. All three services must use the **exact same** secret.

### Find your EC2 public IP

```bash
curl ifconfig.me
```

### Create user-service .env

```bash
cd ~/medical_erp/user-service
cp .env.example .env
nano .env
```

Fill in:

```env
MONGODB_URI=mongodb+srv://admin:<your-password>@cluster0.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority&appName=Cluster0
JWT_SECRET=<paste-generated-secret>
JWT_EXPIRATION=86400000
JWT_REFRESH_EXPIRATION=604800000
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://<your-ec2-public-ip>
```

> **Important**: replace `<your-password>` with your real Atlas password, and append `/users_db` to the URI before the `?`

Save: `Ctrl+O` → Enter → `Ctrl+X`

### Create product-service .env

```bash
cd ~/medical_erp/product-service
cp .env.example .env
nano .env
```

```env
MONGODB_URI=mongodb+srv://admin:<your-password>@cluster0.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority&appName=Cluster0
JWT_SECRET=<same-secret-as-above>
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://<your-ec2-public-ip>
LOW_STOCK_THRESHOLD=10
```

### Create order-service .env

```bash
cd ~/medical_erp/order-service
cp .env.example .env
nano .env
```

```env
MONGODB_URI=mongodb+srv://admin:<your-password>@cluster0.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority&appName=Cluster0
JWT_SECRET=<same-secret-as-above>
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://<your-ec2-public-ip>
PRODUCT_SERVICE_URL=http://localhost:8082/api/v1
USER_SERVICE_URL=http://localhost:8081/api/v1
```

### Create frontend .env.local

```bash
cd ~/medical_erp/frontend
nano .env.local
```

```env
VITE_USER_SERVICE_URL=/api/user
VITE_PRODUCT_SERVICE_URL=/api/product
VITE_ORDER_SERVICE_URL=/api/order
```

---

## 7. Build the Backend

Build each service one at a time. Each must show `BUILD SUCCESS` before moving on.

```bash
cd ~/medical_erp/user-service
mvn clean package -DskipTests
```

Wait for `BUILD SUCCESS`, then:

```bash
cd ~/medical_erp/product-service
mvn clean package -DskipTests
```

Wait for `BUILD SUCCESS`, then:

```bash
cd ~/medical_erp/order-service
mvn clean package -DskipTests
```

Confirm all three jars exist:

```bash
ls ~/medical_erp/user-service/target/*.jar
ls ~/medical_erp/product-service/target/*.jar
ls ~/medical_erp/order-service/target/*.jar
```

---

## 8. Run Backend as Services

This runs the services in the background permanently — they restart automatically on reboot or crash.

> **Critical**: Check if you are logged in as `root` or `ubuntu`:
> ```bash
> whoami
> ```
> If it shows `root`, use `User=root` and paths `/root/medical_erp/...` in all service files below.
> If it shows `ubuntu`, use `User=ubuntu` and paths `/home/ubuntu/medical_erp/...`

The examples below use `root`. If you are `ubuntu`, replace `root` with `ubuntu` everywhere.

### user-service

```bash
sudo nano /etc/systemd/system/user-service.service
```

```ini
[Unit]
Description=User Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/medical_erp/user-service
EnvironmentFile=/root/medical_erp/user-service/.env
ExecStart=/usr/bin/java -jar target/user-service-1.0.0.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### product-service

```bash
sudo nano /etc/systemd/system/product-service.service
```

```ini
[Unit]
Description=Product Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/medical_erp/product-service
EnvironmentFile=/root/medical_erp/product-service/.env
ExecStart=/usr/bin/java -jar target/product-service-1.0.0.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### order-service

```bash
sudo nano /etc/systemd/system/order-service.service
```

```ini
[Unit]
Description=Order Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/medical_erp/order-service
EnvironmentFile=/root/medical_erp/order-service/.env
ExecStart=/usr/bin/java -jar target/order-service-1.0.0.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Start all three

```bash
sudo systemctl daemon-reload
sudo systemctl enable user-service product-service order-service
sudo systemctl start user-service product-service order-service
```

### Verify they are running (wait 30 seconds first)

```bash
sleep 30
curl http://localhost:8081/api/v1/actuator/health
curl http://localhost:8082/api/v1/actuator/health
curl http://localhost:8083/api/v1/actuator/health
```

All three must return `{"status":"UP"}` before continuing.

If any shows `failed`, check logs:
```bash
journalctl -u user-service -n 50 --no-pager
```

The most common cause is a wrong MongoDB URI or password in the `.env` file.

---

## 9. Build and Deploy Frontend

```bash
cd ~/medical_erp/frontend
npm install
npx vite build
```

> Use `npx vite build` NOT `npm run build` — the default script runs a strict TypeScript check that fails on harmless warnings in this codebase.

Copy to Nginx web root:

```bash
sudo mkdir -p /var/www/erp
sudo cp -r dist/* /var/www/erp/
```

---

## 10. Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/erp
```

Paste the entire block below:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/erp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/user/ {
        proxy_pass http://localhost:8081/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/product/ {
        proxy_pass http://localhost:8082/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/order/ {
        proxy_pass http://localhost:8083/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/erp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
```

`nginx -t` must say `syntax is ok` and `test is successful`. If not, re-check the config for typos.

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

Verify everything works:

```bash
curl http://localhost/
curl http://localhost/api/user/actuator/health
curl http://localhost/api/product/actuator/health
curl http://localhost/api/order/actuator/health
```

First should return HTML. Others should return `{"status":"UP"}`.

---

## 11. Bootstrap Users and Orgs

This is the most important step. Run the seed script — it creates all organizations and default users automatically:

```bash
cd ~/medical_erp
bash seed.sh
```

Wait for it to finish. It will print a table like this:

```
============================================
  ✅ MedERP Bootstrap Complete!
============================================

  Open: http://xx.xx.xx.xx

  Login credentials:
  ┌─────────────┬──────────────────────────┬────────────┐
  │ Role        │ Email                    │ Password   │
  ├─────────────┼──────────────────────────┼────────────┤
  │ ADMIN       │ admin@mederr.com         │ Admin@1234 │
  │ DISTRIBUTOR │ distributor@mederr.com   │ Dist@1234  │
  │ HOSPITAL    │ hospital@mederr.com      │ Hosp@1234  │
  └─────────────┴──────────────────────────┴────────────┘
```

> The seed script is safe to run multiple times — it skips anything that already exists.

---

## 12. Test the Application

Open `http://<your-ec2-public-ip>` in your browser.

### Full workflow test

**Step 1 — Add a product and stock (as Distributor)**

1. Log in as `distributor@mederr.com` / `Dist@1234`
2. Go to **Products** → click **+ Add Product**
3. Fill in SKU, Name, Manufacturer, Category, Unit, MRP, Wholesale Price → **Save Product**
4. Add stock via the API (required before orders can be approved):

```bash
# Get distributor token
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"distributor@mederr.com","password":"Dist@1234"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")

# Get product ID
PRODUCT_ID=$(curl -s http://localhost:8082/api/v1/products \
  -H "Authorization: Bearer $TOKEN" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['content'][0]['id'])")

DIST_ORG_ID=$(curl -s -X POST http://localhost:8081/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"distributor@mederr.com","password":"Dist@1234"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['user']['organizationId'])")

# Add 1000 units of stock
curl -X POST http://localhost:8082/api/v1/products/inventory \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"productId\": \"$PRODUCT_ID\",
    \"warehouseId\": \"WH-01\",
    \"warehouseLocation\": \"Main Warehouse\",
    \"batchNumber\": \"BATCH-001\",
    \"manufacturingDate\": \"2026-01-01\",
    \"expiryDate\": \"2028-01-01\",
    \"quantity\": 1000,
    \"reorderLevel\": 10,
    \"distributorId\": \"$DIST_ORG_ID\"
  }"
```

> Without adding stock, the distributor cannot approve orders — you will get `Insufficient stock` error.

**Step 2 — Place an order (as Hospital)**

1. Log out → Log in as `hospital@mederr.com` / `Hosp@1234`
2. Go to **Orders** → click **+ New Order**
3. Select the distributor, pick a product, set quantity and shipping address → **Place Order**

**Step 3 — Approve the order (as Distributor)**

1. Log out → Log in as `distributor@mederr.com` / `Dist@1234`
2. Go to **Dashboard** → see the order under Pending Approvals → click **Approve**
3. Order status changes to `APPROVED`

---

## 13. Troubleshooting

**Services fail with `status=200/CHDIR`**

This means the `WorkingDirectory` in the systemd file does not match where the repo was cloned. Check:
```bash
whoami          # shows your username
echo $HOME      # shows your home directory
```
Update all three systemd service files — change `User=` and `WorkingDirectory=` to match your actual user and home directory. Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart user-service product-service order-service
```

**Health check returns connection refused**

Services take 20-30 seconds to start. Wait and retry:
```bash
sleep 30
curl http://localhost:8081/api/v1/actuator/health
```

**Health check returns DOWN with MongoDB error**

Your MongoDB URI is wrong. Most common mistakes:
- Password has special characters that need URL encoding
- Missing database name in the URI (must have `/users_db`, `/products_db`, `/orders_db` before the `?`)
- IP not whitelisted in Atlas Network Access

Fix the `.env` file and restart:
```bash
sudo systemctl restart user-service
journalctl -u user-service -n 30 --no-pager
```

**`npm run build` fails with TypeScript errors**

Use `npx vite build` instead — it skips the strict type check:
```bash
npx vite build
```

**Organization creation returns 403**

This is fixed in the current codebase — the `SecurityConfig.java` has been updated to allow public org creation. If you are on an older version of the repo, pull latest:
```bash
git pull origin main
mvn clean package -DskipTests
sudo systemctl restart user-service
```

**Order approval fails with `Insufficient stock`**

You created the product but never added inventory. Run the stock addition curl commands in Step 12 above. Every product needs stock added separately before its orders can be approved.

**Products not showing in New Order modal**

The product was added by a user whose `organizationId` is a hospital, not the distributor org. Always add products while logged in as the **DISTRIBUTOR** account. The modal filters products by the selected distributor's org ID.

**Nginx returns 502 Bad Gateway**

A backend service is not running. Check:
```bash
sudo systemctl status user-service product-service order-service
```

Restart whichever is not running.

**Registration fails with `Organization ID is required`**

The Organization ID field on the register form needs a real MongoDB ObjectId from the database — not a number you make up. Run `bash seed.sh` to auto-create orgs and users, or get an existing org ID via:
```bash
curl http://localhost:8081/api/v1/organizations | python3 -m json.tool
```

---

## Registering Additional Users

The registration form requires a valid **Organization ID** — a 24-character MongoDB ID of an existing organization. Here is how to get it:

### Option A — From the browser (easiest)

1. Log in as `admin@mederr.com`
2. Open browser DevTools → **Console** tab
3. Run:
   ```js
   fetch('/api/user/organizations').then(r=>r.json()).then(console.log)
   ```
4. You will see a list of organizations with their `id` fields — copy the one you want

### Option B — From the terminal

```bash
curl http://localhost:8081/api/v1/organizations | python3 -m json.tool
```

Output example:
```json
[
  {
    "id": "6a43eff65c8b9c0bad601362",
    "name": "City General Hospital",
    "type": "HOSPITAL"
  },
  {
    "id": "6a43f12a5c8b9c0bad601368",
    "name": "ABC Distributors Pvt Ltd",
    "type": "DISTRIBUTOR"
  }
]
```

Copy the `id` of the org you want, then go to `http://<your-ec2-ip>/register` and paste it into the **Organization ID** field.

### Registering via curl (if you prefer CLI)

```bash
curl -X POST http://localhost:8081/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "password": "YourPassword123!",
    "role": "HOSPITAL",
    "organizationId": "<paste-org-id-here>"
  }'
```

Valid roles: `ADMIN`, `DISTRIBUTOR`, `HOSPITAL`

### Creating a new organization before registering

If you need a completely new organization (not one of the seeded ones):

```bash
curl -X POST http://localhost:8081/api/v1/organizations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "New Hospital Name",
    "registrationNumber": "HOSP-002",
    "type": "HOSPITAL",
    "address": {
      "street": "123 Main St",
      "city": "Mumbai",
      "state": "MH",
      "pincode": "400001",
      "country": "India"
    },
    "contactEmail": "contact@newhospital.com",
    "contactPhone": "+91-9876543210",
    "active": true
  }'
```

Copy the `id` from the response and use it to register your user.

> **Note**: `registrationNumber` must be unique across all organizations. If you get `Registration number already exists`, change the number (e.g. `HOSP-003`, `HOSP-004`).

---

## Default Login Credentials (after running seed.sh)

| Role | Email | Password |
|------|-------|----------|
| ADMIN | admin@mederr.com | Admin@1234 |
| DISTRIBUTOR | distributor@mederr.com | Dist@1234 |
| HOSPITAL | hospital@mederr.com | Hosp@1234 |

---

## Security Notes Before Going Live

- Change all default passwords immediately
- Restrict MongoDB Atlas Network Access from `0.0.0.0/0` to your EC2's specific IP
- Restrict EC2 security group port 22 to your IP only
- Add HTTPS using Let's Encrypt certbot once you have a domain
- Never commit `.env` files to git




## Testing the Complete Order Workflow

Follow the steps below whenever you create a **new product** in the application.

### Step 1: Create a Product (Distributor)

1. Login as the Distributor.

   **Email:** `distributor@mederr.com`  
   **Password:** `Dist@1234`

2. Navigate to:

   ```
   Products → Add Product
   ```

3. Fill in the required details:

   - SKU
   - Name
   - Manufacturer
   - Category
   - Unit
   - MRP
   - Wholesale Price

4. Click **Save**.

---

## Step 2: Get the Product ID

Run the following command to list all products:

```bash
curl -s http://localhost:8082/api/v1/products \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Example output:

```json
{
  "id": "6a4cb6f54e8b0f584a4d0af9",
  "name": "crocin"
}
```

Copy the **id** of the product you just created.

---

## Step 3: Add Inventory

Set the Product ID:

```bash
NEW_PRODUCT_ID=<YOUR_PRODUCT_ID>
```

Example:

```bash
NEW_PRODUCT_ID=6a4cb6f54e8b0f584a4d0af9
```

Now add inventory for the product:

```bash
curl -X POST http://localhost:8082/api/v1/products/inventory \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"productId\": \"$NEW_PRODUCT_ID\",
    \"warehouseId\": \"WH-01\",
    \"warehouseLocation\": \"Main Warehouse\",
    \"batchNumber\": \"BATCH-003\",
    \"manufacturingDate\": \"2026-01-01\",
    \"expiryDate\": \"2028-01-01\",
    \"quantity\": 1000,
    \"reorderLevel\": 10,
    \"distributorId\": \"$DIST_ORG_ID\"
  }"
```

If successful, you should receive a response similar to:

```json
{
  "productName": "crocin",
  "quantityAvailable": 1000,
  "status": "IN_STOCK"
}
```

---

## Step 4: Create an Order (Hospital)

1. Logout from the Distributor account.

2. Login as the Hospital.

   **Email:** `hospital@mederr.com`  
   **Password:** `Hosp@1234`

3. Navigate to:

   ```
   Orders → New Order
   ```

4. Select:

   - Distributor
   - Newly created product
   - Quantity
   - Shipping Address

5. Click **Place Order**.

The order status will be **PENDING**.

---

## Step 5: Approve the Order (Distributor)

1. Logout from the Hospital account.

2. Login again as the Distributor.

   **Email:** `distributor@mederr.com`  
   **Password:** `Dist@1234`

3. Navigate to:

   ```
   Dashboard → Pending Approvals
   ```

4. Select the pending order.

5. Click **Approve**.

The order status should change from:

```
PENDING
```

to

```
APPROVED
```

---

## Notes

- Creating a product **does not automatically create inventory**.
- Every newly created product **must have inventory added** before it can be ordered and approved.
- If inventory is not added, the distributor will receive an error similar to:

```json
{
  "detail": "Insufficient stock for product: <product-name>"
}
```

- Repeat **Step 2** and **Step 3** for every new product created through the UI.
