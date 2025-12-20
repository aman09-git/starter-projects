# Serverless User Registration System

### **Step 1: Designed a Serverless Backend Architecture**

- Chose a **serverless approach** to avoid EC2 and server management.
- Decided to use **API Gateway + Lambda + DynamoDB**.
- Ensured the system can auto-scale and stay highly available.

---

### **Step 2: Created a User Registration Backend**

- Implemented a Lambda function to **register users**.
- Accepted user details (email, password, name).
- Applied **password hashing** before storing data.
- Stored user information in DynamoDB.
- Ensured logs are generated for debugging.

**Outcome:**

✔ Users can be safely registered

✔ Sensitive data is protected

---

### **Step 3: Implemented Secure Access Using IAM**

- Created an IAM role for Lambda.
- Allowed only required permissions:
    - DynamoDB access
    - CloudWatch logging
- Avoided hard-coded credentials.

**Outcome:**

✔ Secure, role-based access between services

---

### **Step 4: Exposed Backend Using API Gateway**

- Created a REST API as a public entry point.
- Connected API Gateway to Lambda functions.
- Deployed APIs using stages.
- Enabled external access using HTTP endpoints.

**Outcome:**

✔ Backend is accessible like a real application

---

### **Step 5: Created a Login Authentication Flow**

- Implemented a separate Lambda function for **login**.
- Retrieved user data from DynamoDB.
- Validated credentials using hashed passwords.
- Returned appropriate responses:
    - Login successful
    - Invalid password
    - User not found

**Outcome:**

✔ Authentication logic works correctly

✔ Registration and login responsibilities are separated

---

### **Step 6: Performed End-to-End Testing**

- Registered multiple dummy users.
- Verified data persistence in DynamoDB.
- Tested login for:
    - Valid user
    - Invalid password
    - Non-existing user
- Used `curl` for reliable backend testing.

**Outcome:**

✔ Complete user lifecycle validated

✔ Backend behaves like a production system