# <ins>Slingr.io | veritone</ins>
### setups - go course - and more..
`author: @guidoenr`

---

# Engines | Goland | linux env 

# Debian Environment
* **`make build`** doesn't work for a permissions errors, type: 
```go
sudo chmod 777 /var/run/docker.sock
```

* **CTRL+C** doesn't kill the test at localhost:9090
```go
docker stop $containerId
docker rm $containerId
```


---

## TEST Engine locally inside the Google VM 
* Generate Public keys for ssh
```bash
cd ~/.ssh
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub guido@debian # this copy the keys for guido@debian
```


* If you want to run it locally, a small modification needs to be done in main.go 
in line 200.
```go
Replace:
filePath := config.ImageDirectory + chunkInfo.ChunkUUID
For:
filePath := config.ImageDirectory + "temporary"
```
* Once in the Google VM terminal, run make `up-test` and in another VM terminal window make the request as follows:
```bash
curl "Content-Type: multipart/form-data" -F "startOffsetMS=1000" -F "endOffsetMS=2000" -F "cacheURI=https://storage.googleapis.com/artifacts.veritone-334718.appspot.com/containers/Globo/google-label-video.mp4" -F "payload={\"applicationId\":\"applicationId\",\"recordingId\":\"recordingId\",\"jobId\":\"jobId\",\"taskId\":\"taskId\",\"token\":\"token\",\"mode\":\"mode\",\"libraryId\":\"libraryId\",\"libraryEngineModelId\":\"libraryEngineModelId\",\"veritoneApiBaseUrl\":\"https://api.veritone.com\",\"UseGoogleServiceAccount\":\"true\"}" http://0.0.0.0:8080/process
```




## GRAPHQL

### Get Engine Output
```json
query getEngineOutputBad {
  engineResults(jobId:"22041726_0VKkAZDQ13") {
    records {
      tdoId
      engineId
      startOffsetMs
      stopOffsetMs
      jsondata
      assetId
      userEdited
    }
  }
}
```
### Create Job
```graphql
mutation {
  createJob(
    input: {
      target: { status: "downloaded" }
      tasks: [
        {
          ## Webstream Adapter V3F
          engineId: "9e611ad7-2d3b-48f6-a51b-0a1ba40fe255"
          payload: {
            url: "https://storage.googleapis.com/artifacts.veritone-334718.appspot.com/containers/Globo/google-label-video.mp4"
          }
          executionPreferences: {}
          ioFolders: [{ referenceId: "wsa-output", mode: stream, type: output }]
        }
        {
          ## SI2 Playback segment creator V3F
          engineId: "352556c7-de07-4d55-b33f-74b1cf237f25"
          executionPreferences: { parentCompleteBeforeStarting: true }
          ioFolders: [{ referenceId: "pb-input", mode: stream, type: input }]
        }
        {
          ## SI2 audio/video Chunk creator V3F
          engineId: "8bdb0e3b-ff28-4f6e-a3ba-887bd06e6440"
          payload: {
            ffmpegTemplate: "video"
            customFFMPEGProperties: { chunkSizeInSeconds: "300" }
          }
          executionPreferences: { parentCompleteBeforeStarting: true }
          ioFolders: [
            { referenceId: "si-input", mode: stream, type: input }
            { referenceId: "si-output", mode: chunk, type: output }
          ]
        }
        {
          ## Google - Label Detection
          engineId: "148213eb-ccda-42fc-9ad6-7c5159acc798"
          executionPreferences: { parentCompleteBeforeStarting: true }
          }
          ioFolders: [
            { referenceId: "engine-input", mode: chunk, type: input }
            { referenceId: "engine-output", mode: chunk, type: output }
          ]
        }
        {
          ## Output Writer
          engineId: "8eccf9cc-6b6d-4d7d-8cb3-7ebf4950c5f3"
          executionPreferences: { parentCompleteBeforeStarting: true }
          ioFolders: [{ referenceId: "ow-input", mode: chunk, type: input }]
        }
      ]
      routes: [
        {
          parentIoFolderReferenceId: "wsa-output"
          childIoFolderReferenceId: "pb-input"
        }
        {
          parentIoFolderReferenceId: "wsa-output"
          childIoFolderReferenceId: "si-input"
        }
        {
          parentIoFolderReferenceId: "si-output"
          childIoFolderReferenceId: "engine-input"
        }
        {
          parentIoFolderReferenceId: "engine-output"
          childIoFolderReferenceId: "ow-input"
        }
      ]
    }
  ) {
    id
    targetId
    clusterId
  }
}

```

### job
```graphql
# bad cred
{
  job(id: "22041726_0VKkAZDQ13") {
    id
    targetId
    clusterId
    status
    createdDateTime
    tasks {
      records {
        id
        status
        startedDateTime
        completedDateTime
        engine {
          id
          name
        }
        taskPayload
        taskOutput
      }
    }
  }
}
```

```go

type ClientConnection struct {
  videoClient *video.Client
  storageClient *storage.Client
 }

func getClientConnection (...) (*video.Client, *storage.Client){
  if isGoogleServiceAccount(payload){
    videoClient := newClient("withoutCredentials")
    storageClient := newClient("withoutCredentials")
  } else {
    jsonCreds := getJSONCreds()
    storageClient := newClient(withCredentials())
    videClient := newClient(withCredentials())
  }
}