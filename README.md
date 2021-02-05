# kubert
Implementation of Kubernetes for real-time applications using KVM as real-time hypervisor

----

## Overview
Kubernetes is an open source system for managing [containerized applications]
across multiple hosts. It provides basic mechanisms for deployment, maintenance,
and scaling of applications. 

Nowadays, [real-time applications] assume mission-critical tasks in several cyber-physical 
systems (CPS). One of the most important charactersitics of RT applications is the expected 
deterministic behaviour of the host infrastructure.

Virtualization (containerization technology and infrastructure in general) and real-time
may sound like an oxymoron, but its has been demonstrated that [KVM with the right tunning]
can be used as a [real-time hypervisor] with a very low latency.

The objective of this laboratory is to try to obtain most deterministic behaviour for a
Kubernetes implementation, measure the latency and see if it can manage the workload of 
multipe very demanding real-time applications such as [real-time simulation of smart grids].

We will follow an [upstream implementation Kubernetes via KVM, Vagrant and Ansible], however instead of
Ubuntu 18.04 as the recommended host OS we will use Fedora 30 with a 
[PREEMPT_RT patched Linux kernel], then we will have to do some modifications to the playbooks. 

----

## The infrastructure setup

We will use two twin bare-metal machines with the following specs each one:

* **HPE ProLiant DL380 Gen10 server**
* CPUs: 2 x Intel(R) Xeon(R) Gold 6242R CPU @ 3.10GHz 
* Memory: 2 x 32 GiB ...
* Network cards: 2 x Broadcom BCM57416 NetXtreme-E Dual-Media 10G RDMA Ethernet (with checksum offloading)

In summary two bulls :cow: :cow:

The machines are geographically distributed in two laboratories, but virtually coupled by a 
fiber dedicated channel. So we can say that they are in the same network: 192.168.11.0/24.

Check the network latency notebook for the performance of the channel.

----

## Requirements & setup

### PREEMPT_RT patch

So we will start from a fresh [Fedora 33] installation
``` 
sudo yum update
``` 

For sake of simplicity we will use a [precompiled kernel provided by Planet CCRMA]
``` 
sudo rpm -ivh \
http://ccrma.stanford.edu/planetccrma/mirror/fedora/linux/planetcore/33/x86_64/kernel-rt-core-5.10.2-200.rt20.1.fc33.ccrma.x86_64.rpm \
http://ccrma.stanford.edu/planetccrma/mirror/fedora/linux/planetcore/33/x86_64/kernel-rt-modules-5.10.2-200.rt20.1.fc33.ccrma.x86_64.rpm \
http://ccrma.stanford.edu/planetccrma/mirror/fedora/linux/planetcore/33/x86_64/kernel-rt-modules-extra-5.10.2-200.rt20.1.fc33.ccrma.x86_64.rpm \
http://ccrma.stanford.edu/planetccrma/mirror/fedora/linux/planetcore/33/x86_64/kernel-rt-5.10.2-200.rt20.1.fc33.ccrma.x86_64.rpm
```
and set the rt-kernel as the deafult kernel
```
sudo grubby --set-default /boot/vmlinuz-5.10.2-200.rt20.1.fc33.ccrma.x86_64+rt
``` 
### Hyperthreading
Hyperthreading introduces 'random' latencies, so we will disable the Intel (R) Hyperthreading Options to disable the logical processor cores on processors supporting it. \
Reboot the machine, from the `System Utilities` screen, select `System Configuration > BIOS/Platform Configuration (RBSU) > Processor Options > Intel (R) Hyperthreading Options > Disabled > Save Your Settings`. \
At this point cross your fingers, and continue the booting process selecting the PREEMPT_RT patched Linux kernel at the grub. 

After the booting its done, login and do `uname -a` to verify.

### IRQ balancing
Let's isolate the interrupt requests (IRQ) in a range of CPUs. So let's disable the `irqbalance.service` 
```
sudo systemctl stop irqbalance & sudo systemctl disable irqbalance
```

### Real-time test

The [rt-tests test suite], that contains programs to test various real time Linux features is a submodule of this repository. 
We will constantly run latency tests in serveral stages of this laboratory, mostly using [`cyclictest`] for benchmarking RT behaviour in our implementation.
So let's build it:
```
sudo dnf install numactl numactl-devel git \
make automake cmake gcc g++ kernel-devel
git clone --recursive https://github.com/jpnorenam/kubert.git
cd kubert/rt-tests
git checkout stable/v1.0
make all
sudo make install
``` 
At this point we highly recommend to read the `man cyclictest` pages, and let's run our first test
```
sudo cyclictest --mlockall --smp --nanosleep --priority=90 --interval=200 --histogram=250 --distance=0 --loops=36000000 --quiet > ctout 
```
Grab a coffee :coffee:, this will take 2 hours. 

### Dependencies

Now let's install the dependencies
``` 
sudo dnf group install --with-optional virtualization 
sudo dnf install python3 vagrant ansible
``` 

### Vagrant image
``` 
vagrant box add fedora/cloud-base-33
``` 
----

## Acknowledgment

Thanks to the random wizards of Medium and Github :octocat: for the good ideas, posts and repos referenced.

[real-time applications]: https://rt.wiki.kernel.org/index.php/HOWTO:_Build_an_RT-application
[KVM with the right tunning]: https://lwn.net/Articles/656807/
[containerized applications]: https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/
[real-time hypervisor]: https://doi.org/10.1016/j.sysarc.2020.101709
[real-time simulation of smart grids]: https://github.com/DPsim-Simulator/DPsim
[upstream implementation Kubernetes via KVM, Vagrant and Ansible]: https://github.com/talbotfoundry/k8s-kvm
[Fedora 30]: http://fedora.ip-connect.info/linux/releases/30/Server/x86_64/iso/Fedora-Server-dvd-x86_64-30-1.2.iso
[PREEMPT_RT patched Linux kernel]: https://rt.wiki.kernel.org/index.php/Main_Page
[precompiled kernel provided by Planet CCRMA]: http://ccrma.stanford.edu/planetccrma/software/
[`cyclictest`]: https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start
[rt-tests test suite]: https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/rt-tests#compile-and-install
