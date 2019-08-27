# How To Pass a physical NIC Into a Kata Container
![](https://github.com/Yuval-Ai/documentation/blob/doc-create-NIC-passthrough-and-Kata/use-cases/images/NIC%20passthrough%20Diagram.png)


In this guide we walk through the process of passing a physical NIC into a Kata Container. The "usual" way that a KC (Kata Container) is wired for communication is through a series of bridges and virtual NICs as can be seen in the Image above. 

For some use cases, the container demands a direct link to the physical port of the host device, for example in a situation were the container is required to route high BW traffic without having support for acceleration such as SR-IOV.

#### In this guide:

* [Before you start (Restrictions, Requirements and Assumptions)](#Before-you-start-(Restrictions,-Requements-and-Assumptions))
* [Pass a physical NIC into a Kata Container](#Pass-a-physical-NIC-into-a-Kata-Container)
  * [Part 1 – The Host](#Part-1-–-The-Host)
  * [Part 2 – The Container](#Part-2-–-The-Container)
* [Test Your Setup](#Test-Your-Setup)



# Before you start (Restrictions, Requirements and Assumptions)



# Pass a physical NIC into a Kata Container

## Part 1 – The Host



## Part 2 – The Container  



# Test Your Setup