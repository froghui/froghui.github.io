# 输入输出流处理
-a, --attach=[]            Attach to STDIN, STDOUT or STDERR.(docker create/start/run)
-t, --tty=false            Allocate a pseudo-TTY  (only for docker create)
-i, --interactive=false    Keep STDIN open even if not attached (docker create/start/run)

tty的意思是输出中既有stderr,也有stdout，而且stderr按照err格式打印，stdour按照out格式打印；如果不指定tty，那么stderr,stdout的格式不会经过format。注意这里输出的内容其实都是一样的。

interactive的意思是保留输入流

````go
  execConfig := &ExecConfig{
    // TODO(vishh): Expose '-u' flag once it is supported.
    User: "",
    // TODO(vishh): Expose '-p' flag once it is supported.
    Privileged: false,
    Tty:        *flTty,
    Cmd:        execCmd,
    Container:  container,
    Detach:     *flDetach,
  }

  // If -d is not set, attach to everything by default
  if !*flDetach {
    execConfig.AttachStdout = true
    execConfig.AttachStderr = true
    if *flStdin {
      execConfig.AttachStdin = true
    }
  }
````

# 网络配置 (both parent process in defaunt netowrk namespace and child procss in new network namesapce)
--dns=[]                   Set custom DNS servers
--dns-search=[]            Set custom DNS search domains (Use --dns-search=. if you don't wish to set the search domain)
--add-host=[]              Add a custom host-to-IP mapping (host:ip)
--expose=[]                Expose a port or a range of ports (e.g. --expose=3300-3310) from the container without publishing it to your host
-h, --hostname=""          Container host name
--ip=""                    Ip address of the container (format: <ipaddr>/<subnet>[@default_gateway])
--link=[]                  Add link to another container in the form of <name|id>:alias                               'container:<name|id>': reuses another container shared memory, semaphores and message queues
                               'host': use the host shared memory,semaphores and message queues inside the container.  Note: the host mode gives the container full access to local shared memory and is therefore considered insecure.
--mac-address=""           Container MAC address (e.g. 92:d0:c6:0a:29:33)
--net="bridge"             Set the Network mode for the container
                               'bridge': creates a new network stack for the container on the docker bridge
                               'none': no networking for this container
                               'container:<name|id>': reuses another container network stack
                               'host': use the host network stack inside the container.  Note: the host mode gives the container full access to local system services such as D-bus and is therefore considered insecure.
-P, --publish-all=false    Publish all exposed ports to random ports on the host interfaces
-p, --publish=[]           Publish a container's port to the host
                               format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort
                               (use 'docker port' to see the actual mapping)


# namespace(ipc/pid) & cgroup配置(parent process for cpuset/memory)
-c, --cpu-shares=0         CPU shares (relative weight)
--cpuset=""                CPUs in which to allow execution (0-3, 0,1)
--ipc=""                   Default is to create a private IPC namespace (POSIX SysV IPC) for the container
-m, --memory=""            Memory limit (format: <number><optional unit>, where unit = b, k, m or g)
--memory-swap=""           Total memory usage (memory + swap), set '-1' to disable swap (format: <number><optional unit>, where unit = b, k, m or g)
--pid=""                   Default is to create a private PID namespace for the container
--lxc-conf=[]              (lxc exec-driver only) Add custom lxc options --lxc-conf="lxc.cgroup.cpuset.cpus = 0,1"

# rootfs配置 (child process do the mount after pivot_root)
--device=[]                Add a host device to the container (e.g. --device=/dev/sdc:/dev/xvdc:rwm)
-v, --volume=[]            Bind mount a volume (e.g., from the host: -v /host:/container, from Docker: -v /container)
--volumes-from=[]          Mount volumes from the specified container(s)
--read-only=false          Mount the container's root filesystem as read only

# exec配置
--entrypoint=""            Overwrite the default ENTRYPOINT of the image
--env-file=[]              Read in a line delimited file of environment variables
-e, --env=[]               Set environment variables
--privileged=false         Give extended privileges to this container
-w, --workdir=""           Working directory inside the container

# 其他
--name=""                  Assign a name to the container
--restart=""               Restart policy to apply when a container exits (no, on-failure[:max-retry], always)
--cidfile=""               Write the container ID to the file

# 暂时不清楚的
--cap-add=[]               Add Linux capabilities
--cap-drop=[]              Drop Linux capabilities
--security-opt=[]          Security Options
-u, --user=""              Username or UID

# docker run added
-d, --detach=false         Detached mode: run the container in the background and print the new container ID (docker run)
--sig-proxy=true           Proxy received signals to the process (non-TTY mode only). SIGCHLD, SIGSTOP, and SIGKILL are not proxied.  (docker run)
--rm=false                 Automatically remove the container when it exits (incompatible with -d)
