```bash
cat > complete_lab.sh <<'EOF'
#!/bin/bash

echo "=========================================================="
echo "Google Cloud Challenge Lab Setup Script"
echo "=========================================================="
echo "Please enter the exact values from your lab instructions."
echo ""

read -p "Enter the Region (e.g., us-east4): " REGION
read -p "Enter the Zone (e.g., us-east4-a): " ZONE
read -p "Enter the full Bucket Name (e.g., memories-bucket-qwiklabs-...): " BUCKET
read -p "Enter the Pub/Sub Topic Name (e.g., memories-topic-851): " TOPIC
read -p "Enter the Cloud Function Name (e.g., memories-thumbnail-creator): " FUNCTION
RUNTIME="nodejs22"

PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")

echo ""
echo "=== Configuring Environment ==="
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

echo "=== Enabling Required APIs ==="
gcloud services enable \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  eventarc.googleapis.com \
  pubsub.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  logging.googleapis.com

echo "=== Task 1: Creating Storage Bucket ==="
gcloud storage buckets create "gs://$BUCKET" --location="$REGION"

echo "=== Task 2: Creating Pub/Sub Topic ==="
gcloud pubsub topics create "$TOPIC"

echo "=== Setting up IAM Permissions (Wait 90s for propagation) ==="
gcloud services identity create --service=pubsub.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true
gcloud services identity create --service=eventarc.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true

COMPUTE_SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
PUBSUB_SA="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"
EVENTARC_SA="service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com"
GCS_SA=$(gcloud storage service-agent --project="$PROJECT_ID" 2>/dev/null)

grant() {
  gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="$1" --role="$2" --condition=None >/dev/null 2>&1
}
[ -n "$GCS_SA" ] && grant "serviceAccount:$GCS_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/eventarc.eventReceiver
grant "serviceAccount:$COMPUTE_SA" roles/run.invoker
grant "serviceAccount:$COMPUTE_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/storage.objectAdmin
grant "serviceAccount:$COMPUTE_SA" roles/artifactregistry.writer
grant "serviceAccount:$COMPUTE_SA" roles/logging.logWriter
grant "serviceAccount:$COMPUTE_SA" roles/cloudbuild.builds.builder
grant "serviceAccount:$PUBSUB_SA" roles/iam.serviceAccountTokenCreator
grant "serviceAccount:$EVENTARC_SA" roles/eventarc.serviceAgent

sleep 90

echo "=== Task 3: Creating Cloud Function Files ==="
mkdir -p ~/thumbnail-function && cd ~/thumbnail-function

cat > index.js <<'JS1'
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const sharp = require('sharp');

functions.cloudEvent('FUNCTION_NAME_PLACEHOLDER', async cloudEvent => {
  const event = cloudEvent.data;
  const fileName = event.name;
  const bucketName = event.bucket;
  const bucket = new Storage().bucket(bucketName);
  const topicName = "TOPIC_NAME_PLACEHOLDER";
  const pubsub = new PubSub();

  if (fileName.search("64x64_thumbnail") === -1) {
    const filename_split = fileName.split('.');
    const filename_ext = filename_split[filename_split.length - 1].toLowerCase();
    const filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length - 1);

    if (filename_ext === 'png' || filename_ext === 'jpg' || filename_ext === 'jpeg') {
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      const newFilename = `${filename_without_ext}_64x64_thumbnail.${filename_ext}`;
      const gcsNewObject = bucket.file(newFilename);

      try {
        const [buffer] = await gcsObject.download();
        const resizedBuffer = await sharp(buffer)
          .resize(64, 64, { fit: 'inside', withoutEnlargement: true })
          .toFormat(filename_ext)
          .toBuffer();

        await gcsNewObject.save(resizedBuffer, {
          metadata: { contentType: `image/${filename_ext}` },
        });

        console.log(`Success: ${fileName} → ${newFilename}`);
        await pubsub.topic(topicName).publishMessage({ data: Buffer.from(newFilename) });
        console.log(`Message published to ${topicName}`);
      } catch (err) {
        console.error(`Error: ${err}`);
      }
    }
  }
});
JS1

cat > package.json <<'JSON1'
{
 "name": "thumbnails",
 "version": "1.0.0",
 "description": "Create Thumbnail of uploaded image",
 "scripts": { "start": "node index.js" },
 "dependencies": {
   "@google-cloud/functions-framework": "^3.0.0",
   "@google-cloud/pubsub": "^2.0.0",
   "@google-cloud/storage": "^6.11.0",
   "sharp": "^0.32.1"
 }
}
JSON1

# Inject user variables into the JS file
sed -i "s/TOPIC_NAME_PLACEHOLDER/${TOPIC}/; s/FUNCTION_NAME_PLACEHOLDER/${FUNCTION}/" index.js

echo "=== Deploying Cloud Function (Includes automatic retries for IAM propagation) ==="
deployed=0
for i in 1 2 3 4 5; do
  echo "--- Deploy attempt $i ---"
  if gcloud functions deploy "$FUNCTION" --gen2 --runtime="$RUNTIME" --region="$REGION" --source=. --entry-point="$FUNCTION" --memory=512MiB --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" --trigger-event-filters="bucket=$BUCKET" --trigger-location="$REGION" --quiet; then
    echo "✅ Function deployed successfully!"
    deployed=1
    break
  else
    echo "⏳ Eventarc permissions still propagating. Waiting 60 seconds before retrying..."
    sleep 60
  fi
done

if [ $deployed -eq 0 ]; then
  echo "❌ Deployment failed after 5 attempts. Check the logs above."
  exit 1
fi

echo "=== Task 4: Uploading Test Image ==="
cd ~
curl -sSLf -o travel.jpg "https://storage.googleapis.com/cloud-training/arc101/travel.jpg"
gcloud storage cp travel.jpg "gs://$BUCKET/memories-test.jpg"

echo ""
echo "=========================================================="
echo "🎉 Script Complete!"
echo "Wait 15-30 seconds for the thumbnail to generate in your bucket."
echo "You can now go click 'Check my progress' on all tasks in the lab page."
echo "=========================================================="
EOF
```

bash complete_lab.sh
