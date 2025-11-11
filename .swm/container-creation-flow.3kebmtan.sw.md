---
title: Container creation flow
---
This document describes the container creation process, using input parameters to create a container with an isolated writable filesystem layer. It covers configuration verification, writable layer setup, platform-specific settings, network updates, and container registration, resulting in a ready-to-use container.

```mermaid
flowchart TD
  node1["Starting container creation"]:::HeadingStyle --> node2["Assigning the container's writable layer"]:::HeadingStyle
  node2 --> node3["Creating the writable layer in the storage backend"]:::HeadingStyle
  node3 --> node4["Cleanup and platform-specific setup
(Cleanup and platform-specific setup after writable layer)"]:::HeadingStyle
  node4 --> node5["Update network settings
(Cleanup and platform-specific setup after writable layer)"]:::HeadingStyle
  node5 --> node6["Save container state, register, and log creation
(Cleanup and platform-specific setup after writable layer)"]:::HeadingStyle
  node6 --> node7["Return created container
(Cleanup and platform-specific setup after writable layer)"]:::HeadingStyle

  click node1 goToHeading "Starting container creation"
  click node2 goToHeading "Assigning the container's writable layer"
  click node3 goToHeading "Creating the writable layer in the storage backend"
  click node4 goToHeading "Cleanup and platform-specific setup after writable layer"
  click node5 goToHeading "Cleanup and platform-specific setup after writable layer"
  click node6 goToHeading "Cleanup and platform-specific setup after writable layer"
  click node7 goToHeading "Cleanup and platform-specific setup after writable layer"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Starting container creation

<SwmSnippet path="/daemon/create.go" line="64">

---

In `Daemon.create` we start by fetching the image and verifying configs. Then we create the container object. We call `setRWLayer` next to set up the container's writable layer, which is where container-specific filesystem changes will happen, separate from the base image.

```go
func (daemon *Daemon) create(params types.ContainerCreateConfig, managed bool) (retC *container.Container, retErr error) {
	var (
		container *container.Container
		img       *image.Image
		imgID     image.ID
		err       error
	)

	if params.Config.Image != "" {
		img, err = daemon.GetImage(params.Config.Image)
		if err != nil {
			return nil, err
		}
		imgID = img.ID()
	}

	if err := daemon.mergeAndVerifyConfig(params.Config, img); err != nil {
		return nil, err
	}

	if err := daemon.mergeAndVerifyLogConfig(&params.HostConfig.LogConfig); err != nil {
		return nil, err
	}

	if container, err = daemon.newContainer(params.Name, params.Config, imgID, managed); err != nil {
		return nil, err
	}
	defer func() {
		if retErr != nil {
			if err := daemon.cleanupContainer(container, true); err != nil {
				logrus.Errorf("failed to cleanup container on create error: %v", err)
			}
		}
	}()

	if err := daemon.setSecurityOptions(container, params.HostConfig); err != nil {
		return nil, err
	}

	container.HostConfig.StorageOpt = params.HostConfig.StorageOpt

	// Set RWLayer for container after mount labels have been set
	if err := daemon.setRWLayer(container); err != nil {
		return nil, err
	}

	rootUID, rootGID, err := idtools.GetRootUIDGID(daemon.uidMaps, daemon.gidMaps)
	if err != nil {
		return nil, err
	}
	if err := idtools.MkdirAs(container.Root, 0700, rootUID, rootGID); err != nil {
		return nil, err
	}

	if err := daemon.setHostConfig(container, params.HostConfig); err != nil {
		return nil, err
	}
```

---

</SwmSnippet>

## Assigning the container's writable layer

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start setting RW layer"] --> node2{"Does container have an image ID?"}
    click node1 openCode "daemon/create.go:198:199"
    node2 -->|"Yes"| node3["Retrieve base layer from image"]
    click node2 openCode "daemon/create.go:200:206"
    node2 -->|"No"| node4["Set base layer as empty"]
    click node4 openCode "daemon/create.go:199:206"
    node3 --> node5["Create RW layer with base layer"]
    click node3 openCode "daemon/create.go:201:206"
    node4 --> node5
    node5 --> node6["Assign RW layer to container"]
    click node5 openCode "daemon/create.go:207:211"
    node6 --> node7["Finish successfully"]
    click node6 openCode "daemon/create.go:211:213"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/daemon/create.go" line="198">

---

`setRWLayer` checks if the container has an image and gets its root filesystem chain ID. Then it calls the layer store to create a new writable layer on top of that. This writable layer is where container changes will be stored.

```go
func (daemon *Daemon) setRWLayer(container *container.Container) error {
	var layerID layer.ChainID
	if container.ImageID != "" {
		img, err := daemon.imageStore.Get(container.ImageID)
		if err != nil {
			return err
		}
		layerID = img.RootFS.ChainID()
	}
	rwLayer, err := daemon.layerStore.CreateRWLayer(container.ID, layerID, container.MountLabel, daemon.setupInitLayer, container.HostConfig.StorageOpt)
	if err != nil {
		return err
	}
	container.RWLayer = rwLayer

	return nil
}
```

---

</SwmSnippet>

## Creating the writable layer in the storage backend

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start CreateRWLayer"]
    click node1 openCode "layer/layer_store.go:432:485"
    node1 --> node2{"Does layer name exist?"}
    click node2 openCode "layer/layer_store.go:435:438"
    node2 -->|"Yes"| node3["Return error: name conflict"]
    click node3 openCode "layer/layer_store.go:436:438"
    node2 -->|"No"| node4{"Is parent layer specified?"}
    click node4 openCode "layer/layer_store.go:443:444"
    node4 -->|"Yes"| node5{"Does parent layer exist?"}
    click node5 openCode "layer/layer_store.go:444:447"
    node5 -->|"No"| node6["Return error: parent layer missing"]
    click node6 openCode "layer/layer_store.go:445:447"
    node5 -->|"Yes"| node7["Prepare parent layer"]
    click node7 openCode "layer/layer_store.go:448:449"
    node4 -->|"No"| node7
    node7 --> node8{"Is init function provided?"}
    click node8 openCode "layer/layer_store.go:468:469"
    node8 -->|"Yes"| node9["Initialize mount with init function"]
    click node9 openCode "layer/layer_store.go:469:474"
    node8 -->|"No"| node10["Skip initialization"]
    node10 --> node11["Create read-write layer with driver"]
    click node10 openCode "layer/layer_store.go:475:476"
    node9 --> node11
    click node11 openCode "layer/layer_store.go:476:479"
    node11 --> node12["Save mount information"]
    click node12 openCode "layer/layer_store.go:480:483"
    node12 --> node13["Return new layer reference"]
    click node13 openCode "layer/layer_store.go:484:485"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/layer/layer_store.go" line="432">

---

`CreateRWLayer` first checks if there's a parent layer and retrieves it. It defers releasing the parent if an error happens later. Then it creates a mountedLayer struct. If there's an init function, it calls `initMount` to prepare the mount point. Next, it uses the storage driver to create the writable layer and saves the mount state to track it.

```go
func (ls *layerStore) CreateRWLayer(name string, parent ChainID, mountLabel string, initFunc MountInit, storageOpt map[string]string) (RWLayer, error) {
	ls.mountL.Lock()
	defer ls.mountL.Unlock()
	m, ok := ls.mounts[name]
	if ok {
		return nil, ErrMountNameConflict
	}

	var err error
	var pid string
	var p *roLayer
	if string(parent) != "" {
		p = ls.get(parent)
		if p == nil {
			return nil, ErrLayerDoesNotExist
		}
		pid = p.cacheID

		// Release parent chain if error
		defer func() {
			if err != nil {
				ls.layerL.Lock()
				ls.releaseLayer(p)
				ls.layerL.Unlock()
			}
		}()
	}

	m = &mountedLayer{
		name:       name,
		parent:     p,
		mountID:    ls.mountID(name),
		layerStore: ls,
		references: map[RWLayer]*referencedRWLayer{},
	}

	if initFunc != nil {
		pid, err = ls.initMount(m.mountID, pid, mountLabel, initFunc, storageOpt)
		if err != nil {
			return nil, err
		}
		m.initID = pid
	}

	if err = ls.driver.CreateReadWrite(m.mountID, pid, "", storageOpt); err != nil {
		return nil, err
	}

	if err = ls.saveMount(m); err != nil {
		return nil, err
	}

	return m.getReference(), nil
}
```

---

</SwmSnippet>

<SwmSnippet path="/layer/layer_store.go" line="579">

---

`initMount` creates a special init layer with a '-init' suffix to keep compatibility with graph drivers. It creates the layer, gets a mount point, runs the init function on it, and cleans up if the init fails.

```go
func (ls *layerStore) initMount(graphID, parent, mountLabel string, initFunc MountInit, storageOpt map[string]string) (string, error) {
	// Use "<graph-id>-init" to maintain compatibility with graph drivers
	// which are expecting this layer with this special name. If all
	// graph drivers can be updated to not rely on knowing about this layer
	// then the initID should be randomly generated.
	initID := fmt.Sprintf("%s-init", graphID)

	if err := ls.driver.Create(initID, parent, mountLabel, storageOpt); err != nil {
		return "", err
	}
	p, err := ls.driver.Get(initID, "")
	if err != nil {
		return "", err
	}

	if err := initFunc(p); err != nil {
		ls.driver.Put(initID)
		return "", err
	}

	if err := ls.driver.Put(initID); err != nil {
		return "", err
	}

	return initID, nil
}
```

---

</SwmSnippet>

## Cleanup and platform-specific setup after writable layer

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Start container creation"] --> node2["Create platform-specific settings with Config and HostConfig"]
    node2 -->|"Success"| node3["Update network settings with NetworkingConfig"]
    node2 -->|"Failure"| node9["Remove mount points and return error"]
    node3 -->|"Success"| node4["Save container state to disk"]
    node3 -->|"Failure"| node9
    node4 -->|"Success"| node5["Register container"]
    node4 -->|"Failure"| node9
    node5 -->|"Success"| node6["Log container creation event"]
    node5 -->|"Failure"| node9
    node6 --> node7["Return created container"]
    node9["Remove mount points and return error"]

    click node1 openCode "daemon/create.go:121:122"
    click node2 openCode "daemon/create.go:129:131"
    click node3 openCode "daemon/create.go:141:143"
    click node4 openCode "daemon/create.go:145:148"
    click node5 openCode "daemon/create.go:149:151"
    click node6 openCode "daemon/create.go:152:153"
    click node7 openCode "daemon/create.go:153:154"
    click node9 openCode "daemon/create.go:122:126"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/daemon/create.go" line="121">

---

After returning from `setRWLayer` in `Daemon.create`, we set up a deferred cleanup to remove mount points if an error occurs later. Then we apply platform-specific container settings to finalize setup.

```go
	defer func() {
		if retErr != nil {
			if err := daemon.removeMountPoints(container, true); err != nil {
				logrus.Error(err)
			}
		}
	}()

	if err := daemon.createContainerPlatformSpecificSettings(container, params.Config, params.HostConfig); err != nil {
		return nil, err
	}

```

---

</SwmSnippet>

<SwmSnippet path="/daemon/mounts.go" line="20">

---

`removeMountPoints` goes through each mount point in the container. It dereferences volumes and removes them if allowed, but skips named volumes to avoid deleting user-defined persistent data.

```go
func (daemon *Daemon) removeMountPoints(container *container.Container, rm bool) error {
	var rmErrors []string
	for _, m := range container.MountPoints {
		if m.Volume == nil {
			continue
		}
		daemon.volumes.Dereference(m.Volume, container.ID)
		if rm {
			// Do not remove named mountpoints
			// these are mountpoints specified like `docker run -v <name>:/foo`
			if m.Named {
				continue
			}
			err := daemon.volumes.Remove(m.Volume)
			// Ignore volume in use errors because having this
			// volume being referenced by other container is
			// not an error, but an implementation detail.
			// This prevents docker from logging "ERROR: Volume in use"
			// where there is another container using the volume.
			if err != nil && !volumestore.IsInUse(err) {
				rmErrors = append(rmErrors, err.Error())
			}
		}
	}
```

---

</SwmSnippet>

<SwmSnippet path="/daemon/create.go" line="133">

---

After returning from `removeMountPoints` in `Daemon.create`, we update network settings, save the container state to disk, register it with the daemon, and log the creation event.

```go
	var endpointsConfigs map[string]*networktypes.EndpointSettings
	if params.NetworkingConfig != nil {
		endpointsConfigs = params.NetworkingConfig.EndpointsConfig
	}
	// Make sure NetworkMode has an acceptable value. We do this to ensure
	// backwards API compatibility.
	container.HostConfig = runconfig.SetDefaultNetModeIfBlank(container.HostConfig)

	if err := daemon.updateContainerNetworkSettings(container, endpointsConfigs); err != nil {
		return nil, err
	}

	if err := container.ToDisk(); err != nil {
		logrus.Errorf("Error saving new container to disk: %v", err)
		return nil, err
	}
	if err := daemon.Register(container); err != nil {
		return nil, err
	}
	daemon.LogContainerEvent(container, "create")
	return container, nil
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBS3ViZXJuZXRlc01vYnklM0ElM0FHb3BpbmF0aHJlZGR5NjY=" repo-name="KubernetesMoby"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
