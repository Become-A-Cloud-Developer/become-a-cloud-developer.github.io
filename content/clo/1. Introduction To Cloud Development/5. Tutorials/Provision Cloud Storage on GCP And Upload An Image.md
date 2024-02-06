+++
title = "Provision Cloud Storage on GCP And Upload An Image"
weight = 3
date = 2024-02-05
draft = false
+++

## Video

<div style="padding:56.25% 0 0 0;position:relative;"><iframe src="https://player.vimeo.com/video/910241711?h=8e6ce6c6e0&amp;badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" frameborder="0" allow="autoplay; fullscreen; picture-in-picture" style="position:absolute;top:0;left:0;width:100%;height:100%;" title="GCP Cloud Storage"></iframe></div><script src="https://player.vimeo.com/api/player.js"></script>

## Introduction

This tutorial is designed for individuals with a basic understanding of cloud concepts and who are interested in learning how to set up Cloud Storage on Google Cloud Platform (GCP) and upload an image. We will also look into how to make the image publicly available on the Internet as well as how we can keep track on different versions of the file. We will go step-by-step through the process, emphasizing best practices and troubleshooting.

## Method

- We will use the GCP web console to manually provision a Cloud Storage bucket
- We will upload an image to the bucket and make it publicly available
- We will turn on versioning and life cycle management in order to keep track of changes to the file over time

## Prerequisites

- A Google Cloud account. If you don't have one, sign up [here](https://cloud.google.com/).
- Basic knowledge of cloud computing and familiarity with the GCP web console.
- Two different images with the same file name to upload

## Provision a Cloud Storage Bucket

1. Log into Google Cloud Console
	- Go to the [Google Cloud Console](https://console.cloud.google.com/).
	- Log in using your Google account.

2. Create a New Project (if you don't have one already)
	- In the console, go to the project selector page.
	- Click on "New Project" and enter a project name and billing details.

3. Navigate to Cloud Storage
	- With your new project selected, navigate to the "Navigation Menu" (three horizontal lines in the top left corner).
	- Scroll down and select "Storage" under the "Storage" section.

4. Create a Storage Bucket
	- Click on “Create bucket.”
	- Provide a unique name for your bucket. This name has to be globally unique across all Google Cloud Storage.
	- Choose a region for your bucket. For lower latency, choose a region closest to your users.
	- Under "Choose how to control access to objects," select “Uniform” to apply permissions uniformly at the bucket level.
	- Click on “Create.”

5. Enable Versioning on the Bucket
	- After creating your bucket, click on its name to open its details page.
	- Click on the “Configuration” tab.
	- Scroll down to the “Object versioning” section and click on “Edit.”
	- Toggle the “Enable versioning” option to ON.
	- Click “Save” to apply the changes.

Your Cloud Storage bucket is now ready to keep files.

## Upload an Image To The Bucket

1. Navigate back to the “Objects” tab in your bucket’s details page.
2. Click on the “Upload files” button.
3. Select the image file from your computer that you wish to upload and confirm the upload.

## Make the Image Publicly Accessible

1. Go to the "Permissions" tab for the bucket
2. Click on “Grant Access”
3. In the “Add principals” field, enter `allUsers`.
4. Assign the role to “Cloud Storage” and “Storage Object Viewer.”
5. Click “Save.”
6. Confirm "Allow public access"

Your image is now publicly accessible. You can click on the public URL provided to verify that it's accessible in your web browser. You might need to refresh the browser in order to see the change.

## Troubleshooting

- If you encounter issues accessing you image, make sure you use the public URL
- If the wrong version is shown it might be caused by caching the old image in the browser or elsewhere. Change the cache settings.
## Final Thoughts

- Making your data publicly accessible means anyone on the internet can view your files. Always ensure that the data you're making public does not contain sensitive or private information.

## Don't Forget

Google Cloud Storage is a paid service, and charges apply for storage, network usage, and other features like operations and retrieval. Make sure to review the current pricing and manage your resources accordingly.

Don't forget to delete all resources once you are done with this tutorial.

# Happy cloud computing! 🚀