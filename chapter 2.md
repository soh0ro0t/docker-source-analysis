#Router 路由表
------
##00 序
------
Router路由表记录docker cli发送的HTTP请求与docker daemon相应的处理函数的映射关系。

##01 目录
|序号|标题|
|:-:|:-:|
|   1  | container|
|   2  | image|
|   3  | systemrouter|
|   4  | volume|
|   5  | build|

####1 container(r=github.com/docker/docker/api/server/router/container):
        Head request
        request URL                                                     Hander
        /containers/{name:.*}/archive                                   r.headContainersArchive

        Get request
        /containers/json                                                r.getContainersJSON
        /containers/{name:.*}/export                                    r.getContainersExport
        /containers/{name:.*}/changes                                   r.getContainersChanges
        /containers/{name:.*}/json                                      r.getContainersByName
        /containers/{name:.*}/top                                       r.getContainersTop
        /containers/{name:.*}/logs                                      r.getContainersLogs
        /containers/{name:.*}/stats                                     r.getContainersStats
        /containers/{name:.*}/attach/ws                                 r.wsContainersAttach
        /exec/{id:.*}/json                                              r.getExecByID
        /containers/{name:.*}/archive                                   r.getContainersArchive

        Post request
        /containers/create                                              r.postContainersCreate
        /containers/{name:.*}/kill                                      r.postContainersKill
        /containers/{name:.*}/pause                                     r.postContainersPause
        /containers/{name:.*}/unpause                                   r.postContainersUnpause
        /containers/{name:.*}/restart                                   r.postContainersRestart
        /containers/{name:.*}/start                                     r.postContainersStart
        /containers/{name:.*}/stop                                      r.postContainersStop
        /containers/{name:.*}/wait                                      r.postContainersWait
        /containers/{name:.*}/resize                                    r.postContainersResize
        /containers/{name:.*}/attach                                    r.postContainersAttach
        /containers/{name:.*}/copy                                      r.postContainersCopy
        /containers/{name:.*}/exec                                      r.postContainerExecCreate
        /exec/{name:.*}/start                                           r.postContainerExecStart
        /exec/{name:.*}/resize                                          r.postContainerExecResize
        /containers/{name:.*}/rename                                    r.postContainerRename
        /containers/{name:.*}/update                                    r.postContainer Update

        Put request
        /containers/{name:.*}/archive                                   r.putContainersArchive

        Delete request
        /containers/{name:.*}                                           r.deleteContainers

####2 image(r=github.com/docker/docker/api/server/router/image):

        Get request
        /images/json                                                    r.getImagesJSON
        /images/search                                                  r.getImagesSearch
        /images/get                                                     r.getImagesGet
        /images/{name:.*}/get                                           r.getImagesGet
        /images/{name:.*}/history                                       r.getImagesHistory
        /images/{name:.*}/json                                          r.getImagesByName
        Post request
        /commit                                                         r.postCommit
        /images/load                                                    r.postImagesLoad
        /images/create                                                  r.postImagesCreate)
        /images/{name:.*}/push                                          r.postImagesPush)
        /images/{name:.*}/tag                                           r.postImagesTag
        DELETE request
        /images/{name:.*}                                               r.deleteImages

####3 systemrouter(r=github.com/docker/docker/api/server/router/system):

        /{anyroute:.*}                                                  r.optionsHandler
        /_ping                                                          r.pingHandler
        /events                                                         r.getEvents
        /info                                                           r.getInfo
        /version                                                        r.getVersion
        /auth                                                           r.postAuth

####4 volume(r=github.com/docker/docker/api/server/router/volume):

        GET request
        /volumes                                                        r.getVolumesList
        /volumes/{name:.*}                                              r.getVolumeByName
        POST request
        /volumes/create                                                 r.postVolumesCreate
        DELETE request
        /volumes/{name:.*}                                              r.deleteVolumes

####5 build(r=github.com/docker/docker/api/server/router/build):

        /build                                                          r.postBuild
