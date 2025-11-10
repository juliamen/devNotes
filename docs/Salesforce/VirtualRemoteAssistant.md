
# **Visual Remote Assistant (VRA) – Technical Documentation**

## ✅ **Overview**

Visual Remote Assistant (VRA) integrates directly into the Salesforce console, enabling live video sessions from Salesforce objects such as:

* Work Order
* Case
* Care Plan ID
* Lead
* Custom objects

**Key Features:**

* Salesforce Agents deliver interactive AR remote support from Salesforce dashboard.
* Agents and remote experts can see end-user screens and guide resolutions.
* Salesforce information remains accessible during sessions.
* Visual Assistant assets are automatically attached to Salesforce records (e.g., Case).

---

## **1. Installation**

1. Open a browser and go to: [VRA Installation URL](https://sfdc.co/visualremoteassistant-install)
2. Salesforce Login page opens. Log in using your **custom domain** (Communities URL).
3. Select **Install for All Users**.
4. Click **Done**.
5. Verify installation: Setup → Installed Packages → Visual Remote Assistant should be listed.

---

## **2. User Setup**

1. Open the user record that needs VRA access.
2. Click **Permission Set License Assignments** → **Edit Assignment**.
3. Search for **Visual Remote Assistant**, check the box, and click **Save**.
4. Click **Permission Set Assignments** → **Edit Assignment**.
5. Search for **RemoteExpert** permission set → click **Add** → **Save**.

> ⚠️ Note: Remote Expert Permission Set is available only after VRA package installation.

---

## **3. Getting Started**

1. Go to **App Launcher** → search **Visual Remote Assistant – Admin**.
2. The Getting Started page displays:

   * Permission Set Licenses quota for Remote Expert
   * Salesforce Org ID
   * Salesforce Org Type
   * Salesforce Org Name

---

### **3.1 Domain Name Setup**

1. Register a domain under the `techsee.me` primary domain.

2. Enter desired domain → click **Check Availability**.

3. Validation rules:

   | Org Type   | Max Characters |
   | ---------- | -------------- |
   | Production | 10             |
   | Sandbox    | 8              |

4. Uppercase letters are automatically converted to lowercase.

---

### **3.2 Language & Region**

* **Language:** Select End User UI language (Default = English).
* **Region:** Select AWS region:

  * AWS North America (default)
  * AWS Europe

Click **Next** for Prerequisites.

---

## **4. Prerequisites Handling**

### **4.1 Enable Identity Provider**

1. Identity Provider must be enabled for SSO authentication.
2. Click **Enable Identity Provider** → Setup → Identity Provider → Enable → Save.
3. Return to VRA Configuration → check **Identity Provider is enabled** → Next.

---

### **4.2 Connected App Policies**

#### IP Relaxation

* Setup → Manage Connected App → Visual Remote Assistant → Edit Policies → OAuth Policies → IP Relaxation → **Relax IP Restrictions**.

#### Permitted Users

* OAuth Policies → Permitted Users → **Admin approved users are pre-authorized** → Save.
* Return to VRA Configuration → check **Connected App – Policies Edited** → Next.

---

### **4.3 Connected App Profiles**

1. Manage Connected App → Profiles → Manage Profiles.
2. Assign profiles for users who will use VRA.
3. Return to Configuration → check **Connected App – Profiles Assigned** → Next.

---

### **4.4 Custom Attributes**

1. Setup → Manage Connected App → Visual Remote Assistant → Custom Attributes → New.
2. Enter key & value (case-sensitive).
3. Return → check **Connected App – Custom Attributes Created** → Next.

---

### **4.5 SAML Login Information**

1. Setup → Manage Connected App → Visual Remote Assistant → SAML Login Information.
2. Copy:

   * Metadata Discovery Endpoint
   * Single Logout Endpoint
3. Paste in VRA Configuration → check **SAML URLs Copied Successfully** → Click **Create Account**.

---

## **5. Account Activation**

### **5.1 Account Creation**

* Account is created in **Pending state** until Salesforce Admin activates.
* Admin clicks **Create Account** → Verify Prerequisites → **Continue**.

### **5.2 Activation**

* Activate within 48 hours.
* If expired, restart process.
* Successful activation → redirected to **Post Activation Configuration**.

---

## **6. Post-Activation Prerequisites**

### **6.1 Remote Site Settings**

1. Copy API URLs (Visual Remote Support, Visual Remote Image, Stats API).
2. Setup → Remote Site Settings → Create / Edit Remote Sites → Paste URLs.
3. Check **Remote Site Settings Configured** → Next.

> ⚠️ Notification appears if Remote Sites are missing.

---

### **6.2 CSP Trusted Sites**

1. Copy API URLs → Setup → CSP Trusted Sites → Edit → paste → Save.
2. Repeat for all URLs → check **CSP Trusted Sites Configured** → Next.

> ⚠️ Notification appears if CSP Trusted Sites are missing.

---

### **6.3 ACS URL**

1. Copy ACS URL → Manage Connected App → Visual Remote Assistant → Edit Policies → SAML Service Provider Settings → ACS URL → Paste.

---

## **7. Configuration**

### **7.1 Salesforce Object Support**

1. App Launcher → VRA Admin → Configuration → Salesforce Object Support.
2. Add objects to enable VRA (Lead, Case, Work Order, Service Appointment, etc.).
3. Select **Reference Number field** for each object → Save.

### **7.2 Visual Assistant Features**

| Feature                 | Setup Location                                | Action       |
| ----------------------- | --------------------------------------------- | ------------ |
| Audio                   | VRA Configuration → Features → Audio          | Enable Audio |
| Desktop Sharing         | Features → Session Type by Platform → Desktop | Enable       |
| Mobile Screen Mirroring | Features → Mobile                             | Enable       |
| Video Application       | Features → Session Type by Platform → Desktop | Enable       |
| Mobile & Tablet Support | Features → Visual Assistant to-go             | Enable       |

---

### **7.3 Field Service & Feed Tracking**

* Field Service: Setup → Service → Field Service Settings → Enable toggles.
* Feed Tracking: Setup → Feature Settings → Feed Tracking → Enable for supported objects.

---

### **7.4 Visual History**

* Update page layouts to display visual records for each VRA session.

---

## **References**

* [VRA Installation URL](https://sfdc.co/visualremoteassistant-install)
* Salesforce Setup → Installed Packages
* Salesforce Setup → Manage Connected App
* Salesforce App Launcher → Visual Remote Assistant – Admin

---

This Markdown version organizes all your installation, user setup, getting started, prerequisites, activation, post-activation steps, and configuration instructions in a **structured, scannable document**.

---


