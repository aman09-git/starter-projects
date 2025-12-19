# Uploading Website to Amazon S3

**Full Workflow for Hosting a Secure Static Website on S3 + CloudFront + ACM + GoDaddy Domain**

- [https://www.amandevops.shop](https://www.amandevops.shop/?utm_source=chatgpt.com)
- [https://amandevops.shop](https://amandevops.shop/)

# **1️⃣ Uploading Website to Amazon S3**

### **Steps performed**

1. Created S3 bucket:
    
    **portfolio-aman09**
    
2. Uploaded:
    - `index.html`
    - Images, CSS, JS files
3. Disabled:
    - **Block public access OFF**
        
        (so CloudFront can fetch objects)
        
4. Added bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::portfolio-aman09/*"
    }
  ]
}

```

1. Verified public access:
    
    `http://portfolio-aman09.s3.amazonaws.com/index.html` (HTTP)
    

---

# **2️⃣ Setting Up SSL Certificate (ACM)**

### **Steps performed**

1. Opened ACM in **us-east-1** (required for CloudFront)
2. Requested certificate for:
    - `amandevops.shop`
    - `www.amandevops.shop`
3. ACM generated two validation CNAMEs
4. Added CNAME records in **GoDaddy DNS**
5. Verified using `dig`:
    
    ```
    dig _931cf543c94393d910dcd5017bf58138.amandevops.shop cname
    
    ```
    
6. Certificate status changed to **Issued**

---

# **3️⃣ Creating CloudFront Distribution**

### **Steps performed**

1. Open CloudFront → Create Distribution
2. **Origin**: `portfolio-aman09.s3.amazonaws.com`
3. **Viewer Protocol Policy**: Redirect HTTP → HTTPS
4. **Default root object**: `index.html`
5. **Custom SSL Certificate**: Select ACM cert
6. Distribution domain assigned:
    
    `d1tp279mt7rjh4.cloudfront.net`
    
7. Tested CloudFront:
    
    ```
    curl -I https://d1tp279mt7rjh4.cloudfront.net
    
    ```
    
    → Returned HTTP 200
    

---

# **4️⃣ Connecting GoDaddy Domain to CloudFront**

### **Steps performed**

### **A. Nameserver Setup Using Route 53**

1. Created Hosted Zone: `amandevops.shop`
2. Route 53 generated 4 NS values:
    
    ```
    ns-1495.awsdns-58.org.
    ns-229.awsdns-28.com.
    ns-1788.awsdns-31.co.uk.
    ns-619.awsdns-13.net.
    
    ```
    
3. Entered in GoDaddy
    - **Removed trailing dots**
        
        (`.org.` → `.org`)
        
4. Saved → DNS propagation started

---

### **B. Route 53 Records Setup**

1. A Record for Apex Domain:

**Record type:** A – alias

**Name:** (blank → amandevops.shop)

**Target:** CloudFront (d1tp279mt7rjh4.cloudfront.net)

1. A Record for `www`:

**Record type:** A – alias

**Name:** `www`

**Target:** CloudFront

---

### **5️⃣ Final Validation**

1. Checked SSL:
    
    ```
    openssl s_client -connect www.amandevops.shop:443
    
    ```
    
2. Checked HTTP headers:
    
    ```
    curl -I https://amandevops.shop
    curl -I https://www.amandevops.shop
    
    ```
    
3. Final site worked with:
    - HTTPS
    - CloudFront caching
    - Valid SSL