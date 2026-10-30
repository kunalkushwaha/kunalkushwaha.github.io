# Layman guide to Platform Virtualization

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2015-04-25
- Tags: Virtualization, Containers
- Canonical: https://kunalkushwaha.github.io/2015/04/25/layman-guide-to-platform-virtualization/

### Platform Virtualization!

Virtualization is not new in Computer World today.
But when I am asked what I am working on, I say “Cloud Infrastructure and trying to build a private Cloud Platform!”.

Explaining cloud service is easy but Cloud Infrastructure becomes little difficult.
So I am writing a blog series to explain Cloud Infrastructure and it building blocks.
Hope it will help many others too :)

### Virtualization.

Though simple meaning of Virtualization is creating virtual version of anything.
But in computer software world everything is already Virtual :).
But computer hardware devices, Network cable etc are real!

When all these things are created virtually i.e. software simulation of all such hardware its called virtualization!

But why that is required? I heard Hardware is becoming cheap day by day :-|
These are genuine doubts! indeed hardware is becoming cheap and also performance of capacity wise is improving at much faster rate then ever.

Also, with increase of hardware capacity, most of hardware are underutilized :O
Yes and this becomes huge concern for business owner, How to utilize best of your hardware.
Second Management of hardware resources is also a addition of cost.

Using Virtualization both issues can be addressed.
Multiple systems workload is executed on same hardware which result into better utilization,
and Managing multiple systems with software also makes more efficient and easy.

### How is Virtualization is achieved?

There are broadly three types of Virtualization Techniques.
[Wikipedia](http://en.wikipedia.org/wiki/Virtualization#Hardware_virtualization) has short and crisp definition for all three.

Here Host OS is the OS which runs on Actual Hardware and Guest OS is running on Virtual Hardware/Machine.

1. Full virtualization – almost complete simulation of the actual hardware to allow software, which typically consists of a guest operating system, to run unmodified.
   e.g. VirtualBox, VMWare Workstations, Parallel for MAC works on full virtualization, No change is required in guest OS
2. Partial virtualization – some but not all of the target environment attributes are simulated. As a result, some guest programs may need modifications to run in such virtual environments.
   Probably this is first approach to virtualization which lead to full virtualization.
   e.g. IBM mainfraim system [IBM M44/44X] (<http://en.wikipedia.org/wiki/IBM_M44/44X>) was one of such experimental machine.
3. Paravirtualization – a hardware environment is not simulated; however, the guest programs are executed in their own isolated domains, as if they are running on a separate system. Guest programs need to be specifically modified to run in this environment.
   Paravirtulization is lightweight as compared to full virtualization.
   e.g. XEN is based on paravirtualization.

### So what we understood!

Well, We must have realized with all these techniques, all that is achieved here is Isolation!
Isolation of Hardware, so OS running in Virtual Machine gets a feel all hardware is available for its use.
That is the way all OS are implemented! Exclusive access to hardware.

But above three methods are not that efficient with Hardware support for Virtualization.
So, CPU also have extra core and support to run the Virtualization code efficently. More you can find on [here] (<http://en.wikipedia.org/wiki/X86_virtualization>)

Since Isolation is the key to virtulization, Unix & Linux based OS have few features, which provides Isolation to process.
Features like namespaces, cgroups and chroot.
Utilzing these features, OS level virtualization can be achieved. In linux these are called Containers.

I will be writing details of containers in my next blog. So stay tuned :)
