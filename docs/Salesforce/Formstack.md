

# **Formstack Documents – Quick Start Guide (Salesforce)**

## **1. Pre-Installation Steps**

1. **Enable Digital Experiences**

   * Setup → Digital Experiences → Check **Enable Digital Experiences**

2. **Enable CRM Content**

   * Setup → Salesforce CRM Content → Check **Enable Salesforce CRM Content**
   * Check all 4 additional checkboxes after enabling

> ⚠️ For Incommunities projects, only Step 2 may be required.

---

## **2. Installation**

1. Choose your environment:

   * **Production:** [Install Formstack Documents](https://login.salesforce.com/packaging/installPackage.apexp?p0=04t6S000000td8u)
   * **Sandbox:** [Install Formstack Documents](https://test.salesforce.com/packaging/installPackage.apexp?p0=04t6S000000td8u)

2. Complete the installation wizard.

3. Enter your **API Key** and **API Secret** after installation.

---

## **3. Setup a Template**

1. App Launcher → **Formstack Documents** → Open the **Formstack Documents tab**
2. Click **New Document**
3. Enter a **Template Name**
4. Click **Upload File** → select your template

---

## **4. Configure Template Settings**

1. Go to the **Settings** tab → populate relevant settings for your template
2. Go to **Deliver tab** → **Create a Delivery**

   * Select **Salesforce** as the destination
   * Log in with Salesforce credentials
   * Fill in **Record Id** → select where the document should be saved (Attachments or Files)

---

## **5. Map Template Fields**

1. Go to **Formstack Mapping tab** → Click **New Mapping**
2. Select the uploaded template
3. Map the required fields to Salesforce object fields

---

## **6. Troubleshooting / Common Issues**

### **6.1 Reason for Referral Field**

* Issue: Merge field renders as `a;b;c;d;e`
* Solution (Bulleted List):

  1. Create **Case.Formstack_Reason_For_Referral** field
  2. Update IntelliPrint – Outbound flow:

     * Assign `Case.Formstack_Reason_For_Referral` to a local variable
  3. Create a formula field:

     ```text
     SUBSTITUTE("<ul style=list-style-type:disc><li>" & {!changeRFRFormat} & "</ul>", ";", "<li>")
     ```
  4. Update `Case.Formstack_Reason_For_Referral` with formula

---

## **7. Assign Permissions**

Ensure all users who generate Formstack documents have a Permission Set including:

* **Formstack Mappings**
* **Formstack Documents**
* **Formstack Mapping Filter**
* **Formstack Mapping Criteria**
* **Field Mappings**

---

## **8. Notes**

* Always verify **CRM Content** is enabled before installation.
* After installation, make sure **API credentials** are valid.
* Test document generation on a sandbox before deploying to production.

---

This Quick Start Guide condenses the full instructions into **actionable steps**, suitable for admins to follow in order without referring to the full documentation.

---


