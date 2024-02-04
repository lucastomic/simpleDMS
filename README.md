# SimpleDMS
SimpleDMS is a DMS (Document Management Software) based on a very simple architecture and made to meet the challenge provided by IONOS.
## Running and testing
Since this is a repository made with submodules, the way to clone the repository is with the following command

    git clone --recursive https://github.com/lucastomic/simpleDMS
Once cloned, you can boot up the system with the following command

    docker compose up
And the E2E tests can be performed in the `/tests` folder with the following commands

    cd tests
    go test ./...
This will run the next simple E2E tests simulating an user interaction


```go
func TestMetadataSuccessful(t *testing.T) {
	req, _ := helperCreateMultipartRequest(
		t,
		"GET",
		"http://localhost:3001/metadata/file",
		"test.txt",
		"contenido del archivo",
		map[string]string{},
		map[string]string{},
	)
	helperSendRequest(t, req, 200)
}
///...
func TestRetrieveUnexistentFile(t *testing.T) {
	req, _ := helperCreateGETRequest(
		t,
		storageURL,
		map[string]string{},
		// TODO: Instad of using a random number, we should have predefined a constant number, which is realised after testing for future testing.
		[]string{"-1"},
	)
	helperSendRequest(t, req, 404)
}

```
There are also unit tests on the main services `storageService` and `metadataService`, although they have not been done exhaustively.

## Endpoints
### GET /metadata/file
 #### Request 
|  Type | Name | Desc|
|--|--|--|
|  multipart/form-data| uploadFile | file to upload |

 Currently the file information is not necessary, it's requested to simulate that the metadata server returns an upload URL based on the data in this.
 #### Response  
 ```json 
 { 
 "id": 12345, 
 "storageURL": "http://storage-service/upload" 
 }
 ```
 ### POST /storage/file
 #### Request 
|  Type | Name | Desc|
|--|--|--|
|  multipart/form-data| uploadFile | file to upload |
|  text | Id | File's id provided by metadata server |

 #### Response  
 ```json 
{
  "message": "File uploaded successfully."
}

 ```
  ### GET /storage/{id}
 #### Request 
|  Type | Name | Desc|
|--|--|--|
|  path | id | File's ID |

 #### Response  
File requested
 
 
## Architecture

SimpleDMS has the next architecture

![enter image description here](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg-gK6t83mIe-wcGjoPcpGAEe2_cIYKBXExb4v2hmbREDv-tvtgicmuX781oonHM9L26C5uq_G3jBwzMRJ8uLdkr3_wO3zliyV57RqxWYuzvg_wb01OI_lHhRIbdPJEKCUYF6XS4DrQZ8QFXJGy4D2sOiwdeu0Y3R0gKjxWtu3s_LutaimnfJlJS5PfRBQ/s612/SimpleDMS.jpg)

Where APIGateway acts as a reverse proxy, providing the request with a unique request ID that would be maintained throughout the request flow, even if it passes through different services. Thus facilitating the monitoring of logs and traces.

The servers can't be reached directly, due they are in a private network.

As it is believed that this is outside the scope of the task, the APIGateway is a very simple small script that ignores the good practices and other functionalities that it can provide.

``` go
func main() {
	router := mux.NewRouter()
	storageServiceURL, _ := url.Parse(os.Getenv("STORAGE_SERVICE_URL"))
	metadataServiceURL, _ := url.Parse(os.Getenv("METADATA_SERVICE_URL"))
	router.HandleFunc("/storage/{rest:.*}", proxyHandler(storageServiceURL)).
		Methods("GET", "POST", "PUT", "DELETE")
	router.HandleFunc("/metadata/{rest:.*}", proxyHandler(metadataServiceURL)).
		Methods("GET", "POST", "PUT", "DELETE")
	log.Println("API Gateway running on http://localhost:3001")
	log.Fatal(http.ListenAndServe(":3001", router))
}

func proxyHandler(target *url.URL) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		requestID := uuid.New().String()
		r.Header.Add("X-Request-ID", requestID) 
		r.Host = target.Host
		r.URL.Scheme = target.Scheme
		r.URL.Host = target.Host
		r.URL.Path = mux.Vars(r)["rest"]
		proxy := httputil.NewSingleHostReverseProxy(target)
		proxy.ServeHTTP(w, r)
	}
}
```

And following the next 4-layers architecture inside the main services 

![enter image description here](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiig-kXKpscHmC7Nm__Jji1UpVeynho6Yu_MocTl2Dy8_Dgs0hPlyjMiLDcEwwefJteI4uxOVw1asCukrRfsxLRFITm12K2GtK_M2XKXqPf9WZGY_hdcG1XUzrBppm0ogWbwW_Mpqf2iIujBu-YBXQ7mkV_sCB7E9H_VdJJeaskUZatGle6BTmyXoKw8bU/s542/Architeccture.jpg)

Where an attempt is made to maximize the decoupling between each service through abstractions, including the framework used (in this case gorilla/mux) encapsulating it only in the outer layer of Server

## Concurrency
Concurrency has not been used extensively, other than to ensure that IDs will not be repeated in the metadata service in concurrent requests.
```go
func (g *idGenerator) GenerateID() int64 {
	g.mutex.Lock()
	defer g.mutex.Unlock()
	g.lastID++
	return g.lastID
}
```
And for testing the same function
```go
func TestGenerateID_Concurrency(t *testing.T) {
	generator := New()
	var wg sync.WaitGroup
	ids := make(map[int64]bool)
	idsMutex := sync.Mutex{}
	goroutines := 100
	idsPerGoroutine := 10

	wg.Add(goroutines)
	for i := 0; i < goroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < idsPerGoroutine; j++ {
				id := generator.GenerateID()
				idsMutex.Lock()
				if _, exists := ids[id]; exists {
					t.Errorf("Duplicate ID generated: %d", id)
				}
				ids[id] = true
				idsMutex.Unlock()
			}
		}()
	}
	wg.Wait()

	expectedIDs := goroutines * idsPerGoroutine
	if len(ids) != expectedIDs {
		t.Errorf("Expected %d unique IDs, got %d", expectedIDs, len(ids))
	}
}
```

## Logs
Given the execution in containers, I have removed the logs in files such as server.log and api.log and it is only printed in the standard console.
The structure of the logs is as follows, being compatible with applications such as logstash

    time="2024-02-04T21:58:23Z" level=info msg="2024-02-04 21:58:23 | /file | GET | PostmanRuntime/7.36.1 | 200 | 1622" requestID=6cf2c952-a2e7-46b6-a2fd-7f6610109120

Being 1622 the time it took for processing in microseconds.
