# Automate Sitemap Submission to Google Search Console using Google Cloud Functions and Cloud Scheduler

This guide will help you automate the submission of your website's sitemap to Google Search Console using Google Cloud Functions and Cloud Scheduler. Originally developed for a [culinary job board](https://chefjobsnear.me), this guide is particularly useful for job boards, content sites, news sites, blogs, e-commerce platforms, and any website that frequently updates content and wants to ensure timely indexing by Google.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Set Up a Google Cloud Project](#step-1-set-up-a-google-cloud-project)
- [Step 2: Enable Required APIs](#step-2-enable-required-apis)
- [Step 3: Create a Service Account](#step-3-create-a-service-account)
- [Step 4: Prepare the Code](#step-4-prepare-the-code)
  - [index.js](#indexjs)
  - [package.json](#packagejson)
  - [Service Account Key (JSON)](#service-account-key-json)
- [Step 5: Deploy the Cloud Function](#step-5-deploy-the-cloud-function)
- [Step 6: Set Up Cloud Scheduler](#step-6-set-up-cloud-scheduler)
- [Additional Resources](#additional-resources)
- [License](#license)
- [Additional Notes](#additional-notes)

## Overview

Automating sitemap submission ensures that Google is promptly notified of new or updated content on your website. By leveraging Google Cloud Functions and Cloud Scheduler, you can set up a serverless solution that handles this process without manual intervention.

## Prerequisites

- A [Google Cloud Platform (GCP)](https://cloud.google.com/) account
- Access to [Google Search Console](https://search.google.com/search-console/about)
- Basic knowledge of Node.js and JavaScript
- Your website's sitemap URL (e.g., `https://your-website.com/sitemap.xml`)

## Step 1: Set Up a Google Cloud Project

1. Log in to the [Google Cloud Console](https://console.cloud.google.com/).
2. Click on the project dropdown at the top and select **New Project**.
3. Name your project `YOUR-PROJECT` (replace `YOUR-PROJECT` with your desired project name).
4. Click **Create** and wait for the project to be provisioned.

## Step 2: Enable Required APIs

Within your project, you need to enable the following APIs:

- **Google Search Console API**
- **Cloud Functions API**
- **Cloud Scheduler API**

To enable these:

1. Go to **APIs & Services** > **Library**.
2. Search for each API by name.
3. Click on the API and then click **Enable**.

## Step 3: Create a Service Account

1. Navigate to **IAM & Admin** > **Service Accounts**.
2. Click **Create Service Account**.
3. Provide a name like `sitemap-submission-service-account`.
4. Click **Create and Continue**.
5. Assign the role **Owner** (for simplicity in this guide; for production, assign minimal required permissions).
6. Click **Done**.
7. Click on the created service account and navigate to the **Keys** tab.
8. Click **Add Key** > **Create New Key**.
9. Choose **JSON** and click **Create**.
10. Save the JSON key file securely; you'll need it later.

## Step 4: Prepare the Code

Create a new folder for your project and place the following files inside.

### index.js

```javascript
const { google } = require('googleapis');
const { GoogleAuth } = require('google-auth-library');
const path = require('path');

// Define your site URL and sitemap URL
const siteUrl = 'https://your-website.com';
const sitemapUrl = `${siteUrl}/sitemap.xml`;

// Path to the service account key file
const keyFilePath = path.join(__dirname, 'YOUR-SERVICE-ACCOUNT-KEY.json');

// Initialize Google Auth client with the service account
async function initializeAuth() {
  try {
    const auth = new GoogleAuth({
      keyFile: keyFilePath,
      scopes: ['https://www.googleapis.com/auth/webmasters'],
    });
    console.log('GoogleAuth initialized successfully.');
    return auth;
  } catch (err) {
    console.error('Error initializing GoogleAuth:', err);
    throw err;
  }
}

// Function to submit the sitemap
async function submitSitemap(auth) {
  try {
    const webmasters = google.webmasters({
      version: 'v3',
      auth: auth,
    });
    console.log(`Attempting to resubmit sitemap: ${sitemapUrl}`);

    // Submit the sitemap using the Webmasters API
    await webmasters.sitemaps.submit({
      siteUrl: siteUrl,
      feedpath: sitemapUrl,
    });

    console.log(`Sitemap resubmitted successfully: ${sitemapUrl}`);
  } catch (error) {
    console.error('Error during sitemap resubmission:', error.response?.data || error);
    throw error;
  }
}

// Cloud Function's HTTP handler (entry point)
exports.resubmitSitemap = async (req, res) => {
  try {
    console.log('Received request to resubmit sitemap.');
    const auth = await initializeAuth();
    await submitSitemap(auth);

    // Respond with a success message
    res.status(200).send(`Sitemap resubmitted successfully: ${sitemapUrl}`);
  } catch (error) {
    // Log the full error and respond with an error message in case of failure
    console.error('Failed to resubmit sitemap:', error);
    res.status(500).send('Failed to resubmit sitemap. Check logs for details.');
  }
};
```

### package.json

```json
{
  "name": "gsc-sitemap-submission",
  "version": "1.0.0",
  "dependencies": {
    "googleapis": "^39.2.0",
    "google-auth-library": "^7.0.2",
    "@google-cloud/functions-framework": "^3.0.0"
  }
}
```

### Service Account Key (JSON)

Place the JSON file you downloaded in [Step 3](#step-3-create-a-service-account) into your project folder and rename it to `YOUR-SERVICE-ACCOUNT-KEY.json`.

**Important:** Keep this file secure and **never** commit it to a public repository.

## Step 5: Deploy the Cloud Function

1. Go to **Cloud Functions** in the Google Cloud Console.
2. Click **Create Function**.
3. Configure the following settings:
   - **Function Name:** `resubmitSitemap`
   - **Region:** Select a region close to you.
   - **Trigger:** Choose **HTTP**.
   - **Authentication:** Allow **Unauthenticated Invocations**.
4. In the **Runtime** section:
   - **Runtime:** Choose **Node.js 14** (or the latest available).
5. In the **Source Code** section:
   - Choose **ZIP upload**.
   - Click **Browse** and upload a ZIP of your project folder.
6. Set the **Entry Point** to `resubmitSitemap`.
7. Click **Next**, review the settings, and then click **Deploy**.

Wait for the deployment to complete. Once done, you'll have a trigger URL for your Cloud Function.

## Step 6: Set Up Cloud Scheduler

1. Navigate to **Cloud Scheduler** in the Google Cloud Console.
2. Click **Create Job**.
3. Configure the job:
   - **Name:** `sitemap-submission-job`
   - **Frequency:** Set your desired schedule using [cron syntax](https://crontab.guru/). For example, to run every hour: `0 * * * *`.
   - **Time Zone:** Select your time zone.
4. **Target:** HTTP
   - **URL:** Paste the trigger URL of your Cloud Function.
   - **HTTP Method:** `GET`
   - **Auth Header:** **Add OIDC Token**
     - **Service Account Email:** Choose the service account you created earlier.
5. Click **Create**.

Your Cloud Scheduler job is now set up to invoke the Cloud Function at the specified intervals, automating your sitemap submissions.

## Additional Resources

- [Google Cloud Functions Documentation](https://cloud.google.com/functions/docs)
- [Google Cloud Scheduler Documentation](https://cloud.google.com/scheduler/docs)
- [Google Search Console API Reference](https://developers.google.com/webmaster-tools/search-console-api-original/v3/)

## License

This project is licensed under the [MIT License](LICENSE).

## Additional Notes

- **Security:** Ensure your service account has the minimum required permissions in a production environment.
- **Cost Management:** Monitor your Google Cloud usage to avoid unexpected charges.
- **Customization:** You can modify the frequency of the Cloud Scheduler job based on your website's update frequency.

---

*This guide was created to help website owners automate their sitemap submissions, improving SEO efforts by ensuring timely indexing of their content.*
