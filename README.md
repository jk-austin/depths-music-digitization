# Depths Music Digitization
A system to convert physical sheet music to MusicXML. Will provide a user interface to search by a variety of parameters, an admin dashboard to manage digitization, queue review, the upload interface, and a display page for each score, showing metadata, MusicXML viewer, and download options. 

# Architecture:
# Physical Layer:
To produce workable digital image of physical scores, requires:
  - Flatbed scanner or camera rig
  - Standardized capture protocol (300 DPI, consistent lighting, etc.)
  - Produces raw image files (TIFF or high-res JPG)

# Digitization Pipeline:
The heart of the system, to produce MusicXML files, requires:
  - Image Preprocessing Service (Python / OpenCV)
      - Deskewing / noise removal
      - Contrast adjustment
      - Page segmentation (multi-page scores)
  - Quality check — does the image meet minimum quality threshold?
      - Pass → continue
      - Fail → flag for re-scan, add to manual review queue
  - Audiveris OMR Service (as a Java microservice)
      - Converts cleaned image to MusicXML
      - Confidence score returned with output
      - Low confidence → flag for manual review queue
  - Metadata Extraction Service
      - Claude Vision API reads title page
      - Extracts: title, composer, instrumentation, date, publisher
      - Confidence below threshold → flag for manual review queue
  → Validation
      - Human operator reviews flagged items
      - Corrects or approves metadata
      - Approves or rejects MusicXML output

# Job Queue:
Queue managed by Celery (Python-based), under these guidelines:
  - Each score is a job with a status:
      pending → processing → review → complete → failed
  - Failed jobs retry automatically up to 3 times
  - Persistent queue — server restart doesn't lose work
  - Dashboard to monitor throughput and failure rates

# Storage: 
This system will use two storage solutions:
Two storage systems working together:

Object Storage (AWS S3 or equivalent)
  - Raw images (archival)
  - Cleaned images
  - MusicXML files
  - Generated PDFs

Postgres Database (likely Supabase)
  - One record per score:
      - id
      - title
      - composer
      - instrumentation (array)
      - key signature
      - time signature
      - date composed
      - date digitized
      - keywords (array)
      - s3_raw_image_url
      - s3_musicxml_url
      - s3_pdf_url
      - pipeline_status
      - confidence_score
      - manually_reviewed (boolean)
  - Full-text search enabled on title, composer, keywords
  - Filterable by instrumentation, key, time signature

# Backend Services:
Four backend services, each independently deployable:

1. Preprocessing Service (Python)
   - OpenCV image cleanup
   - Feeds cleaned images to Audiveris

2. Audiveris Service (Java based)
   - Standalone OMR microservice
   - Accepts image, returns MusicXML + confidence score

3. Metadata Service (Python)
   - Calls Claude Vision API
   - Returns structured metadata as JSON
   - Falls back to manual entry form if confidence is low

4. API Service (Python / FastAPI or Node.js / Next.js)
   - REST API connecting frontend to database and storage
   - Handles search queries
   - Manages job queue
   - Serves MusicXML and PDF files
  
# Frontend Services:
A Next.js app to provide the following:

  - Search interface
      - Filter by composer, instrumentation, keywords, etc.
      - Full-text search across all metadata
  - Score viewer
      - OpenSheetMusicDisplay renders MusicXML in browser
      - PDF export
  - Admin dashboard
      - Manual review queue for flagged scores
      - Pipeline monitoring (labeling jobs as pending, processing, failed)

# Infrastructure & Deployment:
- Development: Local Docker containers for each service
- Staging: Small cloud instances for the testing pipeline
