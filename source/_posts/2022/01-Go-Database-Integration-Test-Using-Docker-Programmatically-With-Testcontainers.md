---
title: 使用 TestContainers 工具以 Docker 编程方式在 Go 语言中实现数据库集成测试
date: 2022-04-10 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/testcontainers.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Isak Rickyanto
    link: https://rickyanto.com/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，主要介绍了如何使用 Go 语言版本的 TestContainers 工具，以编程的方式实现对 Docker 工具的操作，完成对数据库的集成测试工作。

categories: 
  - 数据库

tags: 
  - Docker
  - TestContainers
---

> 原文链接：https://rickyanto.com/go-database-integration-test-using-docker-programmatically-with-testcontainers/

## 背景介绍

使用 `Docker` 工具进行测试工作已经变得越来与普遍，因为 `Docker` 工具以一种更优雅的方式提供了集成测试和 `e2e` 测试的能力，但是很多开发者仍然依赖 `Docker CLI` 和 `Dockerfile` 工具完成和 `Docker` 的交互工作。

一些开发人员可能会使用 `Makefile` 文件、`bash` 命令来操作 `Docker` 工具；或者在代码中封装 `Docker CLI` 工具的调用，在 `Goland` 语言中可能会使用 `os/exec` 库中的 `exec.Command` 方法进行调用。

我尝试了上面的各种方式，依赖 `Makefile`、`bash` 或者在代码中封装 `Docker CLI` 工具的调用都是不可靠的；在我们的测试工作中，我们无法实现对 `Docker` 工具的优雅集成和进行足够的控制。

所以我尝试去搜索一些其他的实现方案，最终我搜索到了 `testcontainers-go` 这个库，这个库是一个专门的 `Go` 语言依赖库，用于使用 `Docker` 进行测试。这是 [testcontainers](https://www.testcontainers.org/) 项目下的一个子项目，`testcontainers` 项目最初是使用 `Java` 语言构建的。

在我了解了这个工具之后，我突然觉得自己很愚蠢，因为 `Docker` 工具是用 `Go` 语言开发的，并且有 [Docker Engine SDK]((https://docs.docker.com/engine/api/sdk/examples/)) 纯粹用于使用 `Golang` 代码交互和管理 `Docker` 工具。我怎么能把这个给忘了呢，现在已经很明确了，如果我要使用 `Docker` 进行测试，我可以直接使用 `Docker SDK` 调用 `Docker` 工具。但是如果我想使用 `Docker` 的原因只是为了测试目的，相较于直接使用 `SDK` 与 `Docker` 进行交互，我认为使用 `testcontainers-go` 工具很明显是一个更好的选择。

背景已经介绍的足够多了。现在，我将要分享一下我在 `go-starter-kit` 项目是如何使用 `testcontainers-go` 实现一个简单的数据库集成测试的功能的。

## 使用 TestContainers 工具在 Docker 中启动 MySQL数据库

所以我想要做的第一件事是运行提供数据库连接的 `MySQL` `Docker`，并且需要将测试数据填充进去。

我需要实现一个可重用的 `SetupMySQLContainer` 方法，这个方法需要返回 `sqlx.DB` 对象和一个 `function` 对象，`function` 对象将会用于终止 `Docker` `container`。在这个方法实现中我不没有处理数据库凭据问题，因为我要测试的用户存储库不需要 `MySQL` 凭据。

于是我将这个方法放在 `pkg/test` 中的 `db.go` 文件中，因为我希望它能够被其他测试重复使用。

```Go
package test

import (
	"context"
	"fmt"
	"os"

	_ "github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
	"github.com/qreasio/go-starter-kit/pkg/log"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

const (
	dbUsername string = "root"
	dbPassword string = "password"
	dbName     string = "test"
)

var (
	db *sqlx.DB
)

func SetupMySQLContainer(logger log.Logger) (func(), *sqlx.DB, error) {
	logger.Info("setup MySQL Container")
	ctx := context.Background()

	seedDataPath, err := os.Getwd()
	if err != nil {
		logger.Errorf("error get working directory: %s", err)
		panic(fmt.Sprintf("%v", err))
	}
	mountPath := seedDataPath + "/../../test/integration"

	req := testcontainers.ContainerRequest{
		Image:        "mysql:latest",
		ExposedPorts: []string{"3306/tcp", "33060/tcp"},
		Env: map[string]string{
			"MYSQL_ROOT_PASSWORD": dbPassword,
			"MYSQL_DATABASE":      dbName,
		},
		BindMounts: map[string]string{
			mountPath: "/docker-entrypoint-initdb.d",
		},
		WaitingFor: wait.ForLog("port: 3306  MySQL Community Server - GPL"),
	}

	mysqlC, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})

	if err != nil {
		logger.Errorf("error starting mysql container: %s", err)
		panic(fmt.Sprintf("%v", err))
	}

	closeContainer := func() {
		logger.Info("terminating container")
		err := mysqlC.Terminate(ctx)
		if err != nil {
			logger.Errorf("error terminating mysql container: %s", err)
			panic(fmt.Sprintf("%v", err))
		}
	}

	host, _ := mysqlC.Host(ctx)
	p, _ := mysqlC.MappedPort(ctx, "3306/tcp")
	port := p.Int()

	connectionString := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?tls=skip-verify&amp;parseTime=true&amp;multiStatements=true",
		dbUsername, dbPassword, host, port, dbName)

	db, err = sqlx.Connect("mysql", connectionString)
	if err != nil {
		logger.Info("error connect to db: %+v\n", err)
		return closeContainer, db, err
	}

	if err = db.Ping(); err != nil {
		logger.Infof("error pinging db: %+v\n", err)
		return closeContainer, db, err
	}

	return closeContainer, db, nil
}
```

在上面的代码中，你可以看到我首先使用 `testcontainers.ContainerRequest` 准备了 `Docker` 配置信息，然后将其传递给 `testcontainers`。`GenericContainer` 将会启动运行 `Docker` 容器。

然后是 `user_test.go` 代码文件，它是 `Go` 语言实现的 `user` `repository` 集成测试代码，它在 `TestMain` 方法中调用 `SetupMySQLContainer` 函数，因此它可以优雅地启动和终止容器运行。

我将终止容器运行的 `function` 存储在一个变量中，在它内部它将调用 `container.Terminate` 函数。

为了能够填充 `MySQL`（添加测试数据），我需要绑定 `/docker-entrypoint-initdb.d` 到一个文件夹路径，这个文件夹中存储了 `.sql` 文件，这些 `SQL` 文件用户创建 `schema`、`tables` 和添加测试用的数据。代码如下：

```Go
BindMounts: map[string]string{
    mountPath: "/docker-entrypoint-initdb.d",
},
```

在容器被创建之后，下一步我需要获取 `host` 地址和端口来构造 `MySQL DNS` 连接串，以便将其传递给 `sqlx.Connect`。

我觉得这段代码还是比较简单易读的，就不多解释了，大家可以慢慢看，好好理解一下。

## Repository / Database 集成测试

下面是一个简单的集成测试示例，它使用 `SetupMySQLContainer` 方法让 `MySQL` 服务器填充了测试数据并在测试完成后终止它。

```Go
package user_test
 
import (
   "context"
   "os"
   "testing"
 
   _ "github.com/go-sql-driver/mysql"
   "github.com/jmoiron/sqlx"
   "github.com/qreasio/go-starter-kit/internal/user"
   "github.com/qreasio/go-starter-kit/pkg/log"
   "github.com/qreasio/go-starter-kit/pkg/test"
)
 
var (
   db     *sqlx.DB
   logger = log.New()
)
 
func TestMain(m *testing.M) {
   var err error
   var terminateContainer func() // variable to store function to terminate container
   terminateContainer, db, err = test.SetupMySQLContainer(logger)
   defer terminateContainer() // make sure container will be terminated at the end
   if err != nil {
      logger.Error("failed to setup MySQL container")
      panic(err)
   }
   os.Exit(m.Run())
}
 
func TestUserRepository_ListIntegration(t *testing.T) {
   repo := user.NewRepository(db, logger)
   ctx := context.Background()
   req := user.NewListUsersRequest()
   users, err := repo.List(ctx, &amp;req)
 
   if err != nil {
      t.Errorf("error on list users : %s", err)
   }
 
   if len(users) < 1 {
      t.Errorf("Failed to get list users : %s", err)
   }
 
   want := "isak"
   got := users[0].Username
   if got != want {
      t.Errorf("Error get user, want : %s, got : %s", want, got)
   }
}
```

这就是这篇文件的内容。

如果你想了解更多内容，你可以访问 https://www.testcontainers.org 和 https://github.com/testcontainers/testcontainers-go ，如果你想查看上面示例代码文件，你可以访问 https://github.com/qreasio/go-starter-kit ，这是我的 `golang rest api starter kit` 项目，提供可用于新 `API` 项目的初始代码。

## 一个困扰我的问题

正好我需要在一个项目中使用 `testcontainers-go` 这个库，而且也很快就搜到了上面这篇文章，我将作者在 `Github` 上面的 [go-starter-kit](https://github.com/qreasio/go-starter-kit) 仓库代码 `clone` 下来直接运行，一切都是 OK 的，但是当我把相关代码拷贝到我的项目中之后，在运行就失败了，报错如下：

```Go
2022-04-10T18:06:42.271+0800	ERROR	test/testcontainer_mysql.go:72	Error Start MySQL container: failed to create container: Error response from daemon: invalid mount config for type "bind": bind source path does not exist: /docker-entrypoint-initdb.d
github.com/arana-db/arana/test.(*MySQLContainerTester).SetupMySQLContainer
	/Users/dongzonglei/source_code/Github/arana/test/testcontainer_mysql.go:72
github.com/arana-db/arana/test.TestMain
	/Users/dongzonglei/source_code/Github/arana/test/testcontainer_mysql_test.go:46
main.main
```

我查了很多资料，尝试解决这个问题，最后通过这篇文章给出的思路解决了这个问题：[GoLang Postgres Testcontainers Init Script Doesn't Work](https://tutorialmeta.com/question/golang-postgres-testcontainers-init-script-doesnt-work)

我按照文章中介绍的方式调换了 `BindMounts` 参数的顺序，代码如下：

```Go
BindMounts: map[string]string{
    "/docker-entrypoint-initdb.d" : mountPath,
},
```

这次再启动运行，程序运行 OK 了；不过我当时没有搞清楚这是为什么，都是同样的程序，为什么 `map` 参数中 `key & value` 的顺序正好是相反的，不过在写这篇文章的时候我突然想到，会不会是项目依赖的 `testcontainers-go` 版本不同导致存在差异？

顺着这个想法，我首先查看了我的项目中使用的版本：

```
github.com/testcontainers/testcontainers-go v0.12.0
```

使用的是最新的 `v0.12.0` 版本，搜索这个版本的源代码，查看 [BindMounts](https://github.com/testcontainers/testcontainers-go/blob/v0.12.0/docker.go#L746) 参数使用位置：

```Go
// prepare mounts
mounts := []mount.Mount{}
for innerPath, hostPath := range req.BindMounts {
	mounts = append(mounts, mount.Mount{
		Type:   mount.TypeBind,
		Source: hostPath,
		Target: innerPath,
	})
}
```

我们在看一下作者在 `go-starter-kit` 项目使用 `testcontainers-go` 这个库的版本：

```
github.com/testcontainers/testcontainers-go v0.5.1
```

作者使用的是 `v0.5.1` 版本，搜索这个版本的源代码，查看 [BindMounts](https://github.com/testcontainers/testcontainers-go/blob/v0.5.1/docker.go#L543) 参数使用位置：

```Go
// prepare mounts
mounts := []mount.Mount{}
for hostPath, innerPath := range req.BindMounts {
	mounts = append(mounts, mount.Mount{
		Type:   mount.TypeBind,
		Source: hostPath,
		Target: innerPath,
	})
}
```

我们可以看到两个版本中 `BindMounts` 参数中 `map` 集合中 `key & value` 的顺序正好是相反的，这也就解答了上面的问题。

## 后续问题研究

虽然上面通过翻看源码，已经解决了现在的问题，那为什么会调整参数的顺序呢？这个也许通过翻看源码 `Git` 提交记录也许能够找到答案。通过翻看代码的提交记录，找到如下的内容：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/01-Go-Database-Integration-Test-Using-Docker-Programmatically-With-Testcontainers/01_testcontianers_api.png" style="width:800px"/>

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/01-Go-Database-Integration-Test-Using-Docker-Programmatically-With-Testcontainers/02_testcontainers_api.png" style="width:800px"/>

我们通过 Git 提交记录，翻看到对应的 Github 的 PR，在 PR 中作者描述了如下内容：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/01-Go-Database-Integration-Test-Using-Docker-Programmatically-With-Testcontainers/03_testcontainers_api.png" style="width:800px"/>


> 当前无法将本地路径挂载到多个容器路径。BindMounts 当前存储在 map[string]string 集合中，其中 hostPath 是 key， containerPath 是 value。
这可以防止用户将 hostPath 安装到
多个 containerPath。使用 map 结构实际上是有意义的，因为它是不可能的
将多个主机路径挂载到容器中的同一路径。只需要将 map 中键和
值只是交换一下顺序。VolumeMounts 与此相同。

在这里提到了将本地路径映射到容器路径的问题，旧版本的代码中本地路径是 key，容器路径是 value，可以防止用户将多个本地路径映射到同一个容器路径（设置多个只会取到最后一个）；作者也提到了，为了防止这个问题，将 key 和 value 调换一下顺序，通过 map 结构特征，设置多个会被直接覆盖掉。

在最新版本的代码中，这块代码已经被彻底重构掉了，在 `v0.12.0` 版本中，`ContainerRequest` 结构体代码如下：

```Go
// ContainerRequest represents the parameters used to get a running container
type ContainerRequest struct {
	FromDockerfile
	Image           string
	Entrypoint      []string
	Env             map[string]string
	ExposedPorts    []string // allow specifying protocol info
	Cmd             []string
	Labels          map[string]string
	BindMounts      map[string]string
	VolumeMounts    map[string]string
	Tmpfs           map[string]string
	RegistryCred    string
	WaitingFor      wait.Strategy
	Name            string // for specifying container name
	Hostname        string
	Privileged      bool                // for starting privileged container
	Networks        []string            // for specifying network names
	NetworkAliases  map[string][]string // for specifying network aliases
	User            string              // for specifying uid:gid
	SkipReaper      bool                // indicates whether we skip setting up a reaper for this
	ReaperImage     string              // alternative reaper image
	AutoRemove      bool                // if set to true, the container will be removed from the host when stopped
	NetworkMode     container.NetworkMode
	AlwaysPullImage bool // Always pull image
}
```

截止到目前 [7504bdf](https://github.com/testcontainers/testcontainers-go/commit/7504bdf4c99651058e37518698b802ea760291aa) 这个版本的代码，`ContainerRequest` 结构体代码如下：

```Go
// ContainerRequest represents the parameters used to get a running container
type ContainerRequest struct {
	FromDockerfile
	Image           string
	Entrypoint      []string
	Env             map[string]string
	ExposedPorts    []string // allow specifying protocol info
	Cmd             []string
	Labels          map[string]string
	Mounts          ContainerMounts
	Tmpfs           map[string]string
	RegistryCred    string
	WaitingFor      wait.Strategy
	Name            string // for specifying container name
	Hostname        string
	Privileged      bool                // for starting privileged container
	Networks        []string            // for specifying network names
	NetworkAliases  map[string][]string // for specifying network aliases
	NetworkMode     container.NetworkMode
	Resources       container.Resources
	User            string // for specifying uid:gid
	SkipReaper      bool   // indicates whether we skip setting up a reaper for this
	ReaperImage     string // alternative reaper image
	AutoRemove      bool   // if set to true, the container will be removed from the host when stopped
	AlwaysPullImage bool   // Always pull image
	ImagePlatform   string // ImagePlatform describes the platform which the image runs on.
}
```

其中，`BindMounts` 和 `VolumeMounts` 两个 map 结构已经被移除，取代是的 `ContainerMounts` 结构体类型的 `Mounts` 属性，我们来试验一下新的接口该如何调用：

此处暂且留白，最新的代码还没有测试通，网上也没有谷歌到解决办法，等到搞定再补充这块内容。

## 参考链接

- [GoLang Postgres Testcontainers Init Script Doesn't Work](https://tutorialmeta.com/question/golang-postgres-testcontainers-init-script-doesnt-work)
- [testcontainers](https://www.testcontainers.org)
- [testcontainers-go](https://github.com/testcontainers/testcontainers-go)
- [go-starter-kit](https://github.com/qreasio/go-starter-kit)
