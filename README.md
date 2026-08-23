Certainly! I will guide you through deploying the ShopiTry e-commerce application on AWS based on the repository structure and deployment documentation. The main contents of the guide are as follows:

## 🏗️ Application Architecture Overview

```mermaid
flowchart LR
    A[Users] --> B[CloudFront CDN]
    B --> C[S3 Storefront<br/>store.shopitry.com]
    B --> D[S3 Admin Dashboard<br/>admin.shopitry.com]
    B --> E[API Gateway<br/>api.shopitry.com]
    
    E --> F[Gateway Service<br/>EC2 Port 5000]
    
    F --> G[Catalog Service<br/>EC2 Port 5001]
    F --> H[Cart Service<br/>EC2 Port 5002]
    F --> I[Order Service<br/>EC2 Port 5003]
    F --> J[Payment Service<br/>Lambda]
    F --> K[Notification Service<br/>Lambda]
    
    G --> L[(MongoDB Atlas<br/>catalog_db)]
    H --> M[(MongoDB Atlas<br/>cart_db)]
    I --> N[(MongoDB Atlas<br/>orders_db)]
```
### **Quick Navigation**
- [🔄 Phase 1: MongoDB Atlas Configuration](#phase-1-mongodb-atlas-configuration)
- [🔒 Phase 2: AWS Lambda IAM Role](#phase-2-aws-lambda-iam-role)
- [🌐 Phase 3: API Gateway Configuration](#phase-3-api-gateway-configuration)
- [🧩 Phase 4: EC2 Microservices (.env Files)](#phase-4-ec2-microservices-env-files)
- [🚀 Phase 5: CloudFront & ACM Certificate](#phase-5-cloudfront--acm-certificate)
- [🧭 Phase 6: Route 53 DNS Configuration](#phase-6-route-53-dns-configuration)
- [🧪 Phase 7: Final Verification](#phase-7-final-verification)


## 🎯 **Quick Reference: What You Need to Change**

Here's a high-level overview of all the customizations required. We'll dive into each one in detail below.

| Phase | What to Change | Your Actual Value | Why It's Needed |
| :--- | :--- | :--- | :--- |
| **MongoDB Atlas** | IP Access List | Your EC2's public IP | Allows EC2 to connect to Atlas |
| **Lambda Functions** | IAM Role ARN | `arn:aws:iam::YOUR_ACCOUNT_ID:role/...` | Grants Lambda permissions to run |
| **API Gateway** | API ID & Stage | `abc123` (API ID) & `prod` (stage) | Creates HTTP endpoint for Lambda |
| **EC2 Services** | Private IPs of VMs | `10.0.1.2`, `10.0.2.3`, etc. | Allows gateway to route to other services |
| **CloudFront** | ACM Certificate ARN | `arn:aws:acm:us-east-1:...` | Enables HTTPS on your custom domain |
| **Route 53** | Hosted Zone ID & Domain | `Z1D633PEXAMPLE` & `shopitry.com` | Points your domain to CloudFront |
| **Environment Files** | All connection strings & secrets | Your specific URLs, passwords, keys | Configures each service to talk to the right resources |




---

## 🔄 **Phase 1: MongoDB Atlas Configuration**

### **What You Need to Change:**
The IP address that is allowed to connect to your MongoDB Atlas cluster.

### **Where to Find Your Value:**
1. Log into **AWS EC2 Console**
2. Find your **Gateway EC2 instance** (the one that will run the gateway service)
3. Copy its **Public IPv4 address** (e.g., `54.123.45.67`)

### **Where to Make the Change:**
**MongoDB Atlas Console** → **Network Access** tab → **Add IP Address**

### **Before & After:**
```bash
# BEFORE (Default/Example)
Add IP Address: 0.0.0.0/0  # ❌ DANGER: Allows access from anywhere!

# AFTER (Your Actual EC2 Public IP)
Add IP Address: 54.123.45.67  # ✅ Correct: Only your EC2 can connect
```

<details>
<summary>🔍 **Why This Change is Critical**</summary>

If you don't whitelist your EC2's IP, your backend services will fail to connect to the database. This is the **#1 cause of deployment errors** for beginners. Your application will show "MongoNetworkError" or "Connection refused" errors.

**Security Note:** Never use `0.0.0.0/0` in production - this allows any computer on the internet to attempt to connect to your database, which is a major security risk.
</details>

---

## 🔒 **Phase 2: AWS Lambda IAM Role**

### **What You Need to Change:**
The IAM Role ARN that gives your Lambda functions permission to execute.

### **Where to Find Your Value:**
1. Go to **AWS IAM Console** → **Roles**
2. Find the role you created (e.g., `ShopiTryLambdaRole`)
3. Copy the **Role ARN** (looks like `arn:aws:iam::123456789012:role/ShopiTryLambdaRole`)

### **Where to Make the Change:**
In the Lambda deployment commands for both payment-service and notification-service:

### **Before & After:**
```bash
# BEFORE (Example from repository)
--role arn:aws:iam::123456789012:role/service-role/ShopiTryLambdaRole

# AFTER (Your actual IAM role ARN)
--role arn:aws:iam::YOUR_ACCOUNT_ID:role/ShopiTryLambdaRole
```

<details>
<summary>⚙️ **How to Create the IAM Role (If You Don't Have One)**</summary>

1.  Go to **IAM Console** → **Roles** → **Create role**
2.  Choose **AWS service** → **Lambda** as the trusted entity
3.  Attach these **managed policies**:
    *   `AWSLambdaBasicExecutionRole` (for CloudWatch Logs) 【turn0search21】
    *   *(Optional but recommended)* `AmazonS3ReadOnlyAccess` if your Lambda needs S3 access
4.  Give it a name like `ShopiTryLambdaRole` and click **Create role**
5.  Copy the ARN from the Role summary page

**Why This Role is Needed:** Your Lambda function needs permission to write logs to CloudWatch (so you can debug it). The `AWSLambdaBasicExecutionRole` managed policy provides exactly this permission 【turn0search21】.
</details>

---

## 🌐 **Phase 3: API Gateway Configuration**

### **What You Need to Change:**
The API Gateway **Invoke URL** that your backend services will use to call the Lambda functions.

### **Where to Find Your Value:**
1. Go to **API Gateway Console**
2. You should see the API you created (e.g., `ShopiTry-Payment-API`)
3. The ID is a string like `abc123def456` (visible in the API list)
4. The **Invoke URL** is constructed as: `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}` 【turn0search14】

### **Where to Make the Change:**
In the **Gateway Service's `.env` file** on EC2:

### **Before & After:**
```bash
# BEFORE (Example from repository)
PAYMENT_SERVICE_URL=https://<your-api-gateway-id>.execute-api.us-east-1.amazonaws.com/prod/payments/process
NOTIFICATION_SERVICE_URL=https://<your-api-gateway-id>.execute-api.us-east-1.amazonaws.com/prod/notifications/send

# AFTER (Your actual API Gateway Invoke URLs)
PAYMENT_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/payments/process
NOTIFICATION_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/notifications/send
```

> 💡 **Pro Tip:** After creating your API in API Gateway, **deploy it** to a stage named `prod` (or any name you choose). The Invoke URL is only active after deployment. You can test it using the **Test** button in the API Gateway console.

---

## 🧩 **Phase 4: EC2 Microservices (.env Files)**

This is the most detailed section. Each service has its own `.env` file with specific configurations.

### **4.1 Gateway Service Configuration**
**File Location:** `/var/www/shopitry/backend/gateway-service/.env`

### **What You Need to Change:**

| Variable | What to Change It To | Where to Find the Value |
|----------|----------------------|-------------------------|
| `JWT_SECRET` | A strong, random string (min 32 characters) | Generate with: `openssl rand -hex 32` |
| `CATALOG_SERVICE_URL` | `http://<VM_SERVER_2_PRIVATE_IP>:5001` | EC2 Console → Private IP of Catalog service instance |
| `CART_SERVICE_URL` | `http://<VM_SERVER_3_PRIVATE_IP>:5002` | EC2 Console → Private IP of Cart service instance |
| `ORDER_SERVICE_URL` | `http://<VM_SERVER_4_PRIVATE_IP>:5003` | EC2 Console → Private IP of Order service instance |
| `PAYMENT_SERVICE_URL` | `https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/payments/process` | From Phase 3 above |
| `NOTIFICATION_SERVICE_URL` | `https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/notifications/send` | From Phase 3 above |

### **Before & After Example:**
```bash
# BEFORE (Example from repository)
PORT=5000
JWT_SECRET=shopitry_super_secret_jwt_key_2026
CATALOG_SERVICE_URL=http://<VM_SERVER_2_PRIVATE_IP>:5001
CART_SERVICE_URL=http://<VM_SERVER_3_PRIVATE_IP>:5002
ORDER_SERVICE_URL=http://<VM_SERVER_4_PRIVATE_IP>:5003
PAYMENT_SERVICE_URL=http://<LAMBDA_PAYMENT_URL_OR_PORT_5004>
NOTIFICATION_SERVICE_URL=http://<LAMBDA_NOTIFICATION_URL_OR_PORT_5005>

# AFTER (Your actual configuration)
PORT=5000
JWT_SECRET=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6  # Generate your own!
CATALOG_SERVICE_URL=http://10.0.1.2:5001       # Use your actual private IPs
CART_SERVICE_URL=http://10.0.1.3:5002
ORDER_SERVICE_URL=http://10.0.1.4:5003
PAYMENT_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/payments/process
NOTIFICATION_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/notifications/send
```

### **4.2 Catalog Service Configuration**
**File Location:** `/var/www/shopitry/backend/catalog-service/.env`

### **What You Need to Change:**

| Variable | What to Change It To | Where to Find the Value |
|----------|----------------------|-------------------------|
| `MONGODB_URI` | `mongodb+srv://linux:YOUR_PASSWORD@edublitz.laegfsa.mongodb.net/catalog_db?appName=edublitz` | MongoDB Atlas Console → Connect → Your connection string |

### **Before & After Example:**
```bash
# BEFORE (Example from repository)
PORT=5001
MONGODB_URI=mongodb+srv://linux:redhat@edublitz.laegfsa.mongodb.net/catalog_db?appName=edublitz

# AFTER (Your actual MongoDB URI with your password)
PORT=5001
MONGODB_URI=mongodb+srv://linux:YourStrongPassword123!@edublitz.laegfsa.mongodb.net/catalog_db?appName=edublitz
```

<details>
<summary>🔧 **How to Get Your MongoDB Connection String**</summary>

1.  Go to **MongoDB Atlas Console** → **Clusters** → **Connect**
2.  Choose **Connect your application**
3.  Copy the connection string - it looks like:
    ```
    mongodb+srv://linux:<password>@edublitz.laegfsa.mongodb.net/?appName=edublitz
    ```
4.  Replace `<password>` with your actual database user password
5.  Add the specific database name (`catalog_db`, `cart_db`, or `orders_db`) before the `?`

**Important:** If your password contains special characters (like `@`, `#`, `$`), you may need to URL-encode them. For example, `@` becomes `%40`.
</details>

### **4.3 Cart & Order Services Configuration**
The Cart and Order services follow the same pattern as the Catalog service, just with different database names and possibly additional environment variables.

**Cart Service (`/var/www/shopitry/backend/cart-service/.env`):**
```bash
PORT=5002
MONGODB_URI=mongodb+srv://linux:YourPassword123!@edublitz.laegfsa.mongodb.net/cart_db?appName=edublitz
```

**Order Service (`/var/www/shopitry/backend/order-service/.env`):**
```bash
PORT=5003
MONGODB_URI=mongodb+srv://linux:YourPassword123!@edublitz.laegfsa.mongodb.net/orders_db?appName=edublitz
CART_SERVICE_URL=http://10.0.1.3:5002  # Private IP of Cart service
PAYMENT_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/payments/process
NOTIFICATION_SERVICE_URL=https://abc123def456.execute-api.us-east-1.amazonaws.com/prod/notifications/send
```

---

## 🚀 **Phase 5: CloudFront & ACM Certificate**

### **What You Need to Change:**
The **ACM Certificate ARN** that enables HTTPS on your CloudFront distribution.

### **Where to Find Your Value:**
1. Go to **AWS Certificate Manager (ACM) Console**
2. **Important:** Ensure you are in the **US East (N. Virginia)** region (top right corner) - this is a hard requirement from AWS 【turn0search5】
3. Find the certificate you requested for your domain (e.g., `*.shopitry.com` or `store.shopitry.com`)
4. Copy its **ARN** (looks like `arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012`)

### **Where to Make the Change:**
In the **CloudFront distribution settings** under **Viewer certificate**.

### **Before & After:**
```bash
# BEFORE (Example from repository)
ACM Certificate ARN: (not specified in scripts - you must add this)

# AFTER (Your actual ACM certificate ARN)
ACM Certificate ARN: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

<details>
<summary>⚠️ **Critical Certificate Requirement**</summary>

**The ACM certificate MUST be requested in the US East (N. Virginia) region (`us-east-1`)**, even if your CloudFront distribution is in another region. This is a hard requirement from AWS 【turn0search5】【turn0search7】.

If you request the certificate in another region (like `us-west-2`), you won't be able to select it when creating your CloudFront distribution, and you'll see an error like "Certificate not found" or "Invalid certificate".

**How to Request the Certificate:**
1. Go to ACM Console in `us-east-1`
2. Request a public certificate for your domain (e.g., `*.shopitry.com` or `store.shopitry.com`)
3. Validate it (usually via DNS validation - add the CNAME record to your Route 53 hosted zone)
4. Wait for status to change to "Issued" before using it
</details>

---

## 🧭 **Phase 6: Route 53 DNS Configuration**

### **What You Need to Change:**
The **Alias Target** in your Route 53 hosted zone record.

### **Where to Find Your Value:**
1. Go to **CloudFront Console**
2. Find your distribution (e.g., for storefront)
3. Copy the **Domain Name** (looks like `d111111abcdef8.cloudfront.net`)

### **Where to Make the Change:**
**Route 53 Console** → **Hosted zones** → select your domain → **Create Record**

### **Before & After:**
```bash
# BEFORE (Example from repository)
Record name: store
Record type: A
Alias: Yes
Route traffic to: Alias to CloudFront distribution
Value: d111111abcdef8.cloudfront.net  # Example CloudFront domain

# AFTER (Your actual CloudFront distribution domain)
Record name: store
Record type: A
Alias: Yes
Route traffic to: Alias to CloudFront distribution
Value: d1234567890.cloudfront.net  # Your actual CloudFront domain
```

> 💡 **Important:** Always select the CloudFront distribution from the dropdown list instead of manually entering the domain name. This ensures the correct configuration and automatically handles future changes.

---

## 🧪 **Phase 7: Final Verification**

After making all changes, use this checklist to verify everything is configured correctly:

### **Quick Verification Commands:**
```bash
# 1. Test Gateway Health Endpoint (should return 200 OK)
curl -i https://api.shopitry.com/health

# 2. Test Catalog Products API (should return JSON data)
curl -i https://api.shopitry.com/api/catalog/products

# 3. Test Admin Authentication (should return JWT token)
curl -i -X POST https://api.shopitry.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@shopitry.com","password":"adminpassword123"}'
```

### **Common Issues & Solutions:**

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| **MongoNetworkError** | EC2 IP not whitelisted in Atlas | Add EC2 public IP to MongoDB Atlas Network Access 【turn0search19】 |
| **Lambda timeout** | Incorrect IAM role or permissions | Verify Lambda execution role has `AWSLambdaBasicExecutionRole` 【turn0search21】 |
| **API Gateway 403** | API not deployed or incorrect URL | Deploy API to a stage and use correct Invoke URL format |
| **CORS errors** | Missing CORS headers in API Gateway | Configure CORS in API Gateway methods and enable OPTIONS |
| **CloudFront 502** | Origin (S3 bucket) not accessible | Check S3 bucket policy and ensure it allows CloudFront access |

---

## 📊 **Configuration Summary Table**

Here's a quick reference of all values you need to change:

| Phase | What to Change | Example Value | Where to Find |
|-------|----------------|---------------|---------------|
| **MongoDB Atlas** | IP Access List | `54.123.45.67` | EC2 Console → Public IP |
| **Lambda IAM** | Role ARN | `arn:aws:iam::123456789012:role/ShopiTryLambdaRole` | IAM Console → Roles |
| **API Gateway** | API ID | `abc123def456` | API Gateway Console |
| **Gateway .env** | JWT_SECRET | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` | Generate with `openssl rand -hex 32` |
| **Gateway .env** | Private IPs | `10.0.1.2`, `10.0.1.3`, `10.0.1.4` | EC2 Console → Private IPs |
| **Service .env** | MongoDB URI | `mongodb+srv://linux:password@edublitz...` | Atlas Console → Connect |
| **CloudFront** | ACM Certificate ARN | `arn:aws:acm:us-east-1:...` | ACM Console (us-east-1) |
| **Route 53** | Alias Target | `d1234567890.cloudfront.net` | CloudFront Console |

---

## 🎯 **Beginner Tips for Success**

1.  **Use a Text Editor with Syntax Highlighting**: When editing `.env` files, use an editor that highlights syntax (like VS Code) to avoid typos.
2.  **Test Incrementally**: Don't try to configure everything at once. Set up and test each component (MongoDB, then Lambda, then EC2 services) before moving to the next.
3.  **Use AWS CLI for Automation**: Once you understand the manual process, consider using AWS CloudFormation or Terraform to automate the deployment.
4.  **Check Logs Immediately**: If something doesn't work, check the logs immediately:
    *   **EC2 Services**: `pm2 logs gateway-service`
    *   **Lambda**: CloudWatch Logs → Log groups → `/aws/lambda/shopitry-payment-service`
    *   **API Gateway**: Use the **Test** button in the API Gateway console
5.  **Keep Security in Mind**: Never commit `.env` files to version control. Use `.env.example` files as templates and keep your actual `.env` files secure.

---

## 🆘 **Getting Help**

If you get stuck:

1.  **Check AWS Service Logs**: CloudWatch Logs for Lambda and API Gateway, EC2 system logs
2.  **Verify Security Groups**: Ensure EC2 security groups allow necessary traffic
3.  **Test Connectivity**: From your EC2 instance, try to ping other services and access MongoDB
4.  **Use AWS Forums**: AWS has re:Post forums where experts can help
5.  **Check Repository Issues**: The GitHub repository may have known issues or solutions

Remember: **Every configuration value has a specific source**. When in doubt, go back to the source (AWS console, MongoDB Atlas, etc.) to verify you're using the correct value.

> 🎉 **Congratulations!** By following this guide, you've successfully configured all the necessary values for your ShopiTry deployment on AWS. Take your time, double-check each value, and don't hesitate to use the debugging tips if something doesn't work on the first try.