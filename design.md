# Raspberry Pi Remote Camera Control Project Design Document
- By Justyce Hickman 10/1/2026

# Overview

1. Given a Raspberry Pi Computer; we will connect it to a power source and a usb camera; for field testing we will use a portable battery with sufficient power and energy to make the pi run a couple hours and a smaller compact camera
2. When field testing is the highest priority we will use cad software to build a case to hold the raspberry pi, portable battery and camera in a safe and efficient way for protecting the parts and recording using the device
3. The raspberry pi will have software to automatically run a program as soon as it boots up; this software will enable the server that allows us to request to the pi; it will start a hotspot so we can use our phone to connect to the pi directly; it will wait for request from the phone and then trigger recordings
4. The raspberry pi will function when connected to a common wifi network, if connected to a common wifi network we will use the raspberry pi's ip or some forwarding on that network to allow us to utilize a web page to access the control button for triggering recording. If it is connected to a common wifi network it will also trigger/signal/or send request to the software intensive apart of this project for uploading these files to the cloud, and saving them in a database to be later accessed whether they want to be downloaded or viewed from a website dashboard
5. The actual uploading of the videos and files should use aws services like S3 and lambda for project complexity and learning, the uploading process should be viewable through the web dashboard as well
6. On the raspberry pi boot, if not connected to an external display like a monitor there should be some way to validate that the program is running correctly, this can be done using the web dashboard or some other checks; it should report whether the hotspot is running, camera is connected, whether you are able to record, server status, and storage status
7. Uploading to the cloud should not delete the files from the pi, after a successfull upload has occured we can mark those videos with successfully uploaded and then delete them from the pi if storage is close to running out or i decide to manually delete them
8. Each recording session should last 3 minutes once the button has been clicked, however we will provide a stop button that can stop it earily, button clicks within a 3 minute timer of a already started recording should not end the recording at that moment but prompt to click the stop button first, save it and start a new recording if they clicked stop and then start. Quality of the capture should be prioritized regardless 30fps and 1080p

# Goals

- Raspberry pi allows for on command recording, safely stores videos, uses apis, servers, and aws services for video management automatically once connected to power source
- Allows for remote control for 20-50ft range
- Has field mode and sync modes, able to switch modes when on wifi connection
- Designed case is simple and safe for storing parts
- Any of the software services can be turned off at anytime, project can be recreated easily through clear instructions of set up through readme
- Document each particularly changing step of development
- Use AI assisted coding, object oriented programming, c++. Have AI speed up programming workflow as fast as possible, but remember important concepts, add comments to alot of the code explaining anything i dont understand so i can come back later

# Risk and Open Questions
- Dont know how long the raspberry pi hotspot range is
- Dont know how to use aws services
- Dont know how to use cad software
- Dont know how raspberry pi will work with battery
- Dont know how weather conditions will effect it
- Heat and designs

# Software

- AWS S3
- AWS LAMBDA
- DynamoDB
- API gateway and Lambda
- Next.js & React for frontend
- C++ (for pi)
- systemd
- Network Manager
- CloudWatch
- Github actions
# Tech

- Raspberry pi
- Usb camera
- Portable charger

# Data Flow

## In the field (no internet)
1. User powers on the Pi. systemd starts the hotspot, local web server, and recorder service. The Pi runs boot health checks (camera detected, storage free, hotspot up) and exposes the results at GET /status.
2. User joins the Pi's hotspot and opens the control page (e.g. http://192.168.4.1). The page shows the status from /status.
3. User presses Start. The phone sends POST /record/start. The Pi creates a new recording with a unique ID (e.g. 2026-10-01T14-30-00_a1b2) and starts the 3-minute timer.
4. The recorder captures 1080p30 video and writes it as 10-second .ts chunks to persistent storage:
   recordings/<recording_id>/chunk_0000.ts, chunk_0001.ts, ...
5. When each chunk finishes, it is flushed to disk (fsync) and added to the upload queue (SQLite on disk) with status: pending.
6. The recording ends when the user presses Stop (POST /record/stop) or the 3-minute timer runs out. Pressing Start during a recording returns a message telling the user to press Stop first.
7. When the recording ends, the Pi writes manifest.json for it: recording ID, start/end time, chunk count, and a checksum for each chunk. The manifest is also added to the queue, marked to upload last.
8. User shuts the Pi down cleanly from the control page (or the power button) and takes it home.

## Back home (sync)
9. On boot, the Pi checks for known home Wi-Fi. If it's in range, the Pi joins it as a client (sync mode); otherwise it starts its own hotspot (field mode). The control page is reachable at the Pi's LAN address (e.g. http://campi.local).
10. The uploader reads the queue from disk and uploads pending chunks to S3 in order:
    s3://<bucket>/recordings/<recording_id>/chunk_0000.ts
    Each upload sends the chunk's checksum, so S3 rejects it if the data arrived corrupted.
11. When S3 confirms a chunk, the queue marks it uploaded. Failed uploads stay pending and are retried with exponential backoff (wait 1s, 2s, 4s, ...).
12. Each chunk landing in S3 triggers the processing Lambda, which:
    - writes the chunk's metadata to DynamoDB (recording ID, chunk number, size, upload time)
    - updates the recording's HLS playlist (playlist.m3u8) so the chunks uploaded so far can be streamed
13. After all of a recording's chunks are confirmed, the Pi uploads manifest.json.
14. The manifest triggers the stitch Lambda, which:
    - compares the manifest's chunk list against the chunks recorded in DynamoDB
    - if any are missing, marks the recording "incomplete" and stops (the Pi will retry those chunks)
    - if all are present, joins the chunks into one MP4 without re-encoding, saves it to S3, and marks the recording "complete"
15. The dashboard (Next.js) calls the API (API Gateway + Lambda), which reads DynamoDB to list recordings with their status and upload progress (chunks uploaded / total).
16. Viewing: the API returns a short-lived presigned URL for the HLS playlist, and the browser streams it.
    Downloading: the API returns a presigned URL for the stitched MP4.
17. Local cleanup: uploaded chunks stay on the Pi until storage runs low or the user deletes them manually. Only chunks marked "uploaded" can ever be deleted.


# Failure Cases

| What fails | What happens | How the system recovers |
|---|---|---|
| Battery dies mid-recording | The chunk being written is lost or corrupted; finished chunks are safe on disk | On boot, the queue reloads from SQLite; incomplete chunk files are discarded; a manifest is written for the partial recording so it can still be processed |
| Recorder crashes mid-recording | Recording stops unexpectedly | systemd restarts the service; on startup it finds the unfinished recording, closes it out with a manifest, and returns to idle (it does not auto-resume recording) |
| Camera unplugged or not detected | Recording can't start or stops | /status reports "camera missing"; Start is disabled on the control page; the Pi re-checks for the camera every few seconds |
| SD card full | New chunks can't be written | The recording stops cleanly and the status page shows a warning; uploaded chunks are deleted oldest-first to free space; un-uploaded chunks are never auto-deleted |
| Wi-Fi drops mid-upload | The chunk upload fails or times out | The chunk stays "pending" in the queue and is retried with exponential backoff when the connection returns |
| Chunk corrupted in transit | Data doesn't match the checksum | S3 rejects the upload; the chunk stays pending and is retried |
| Same chunk uploaded twice (e.g. S3 confirmed but the Pi lost the response) | Duplicate upload attempt | Same S3 key, so the file is just overwritten; Lambda writes metadata keyed by recording ID + chunk number, so no duplicate rows (idempotent) |
| Manifest arrives but chunks are missing | Recording can't be stitched | Stitch Lambda marks the recording "incomplete"; when the missing chunks arrive, the chunk Lambda re-checks and triggers stitching |
| Stitch Lambda fails (timeout, error) | No downloadable MP4 | Recording stays "processing"; the error is logged in CloudWatch and triggers an alarm; stitching can be retried, and streaming through HLS still works |
| Phone loses connection to the hotspot | User can't see status or press Stop | Recording continues; the 3-minute timer ends it automatically; on reconnect, the page reloads the current state from /status |
| Pi overheats | CPU slows down; frames may drop | /status reports temperature and throttling; logged so heat can be measured during field tests |
| AWS costs spike unexpectedly | Bill goes up | A billing alarm emails at a set threshold |
