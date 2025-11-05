---
published: true
date: '2025-11-05 16:22 +0100'
title: 2025-11-05-how-to-solve-the-AI-fiber-capacity-crunch
author: Maurizio Gazzola
excerpt: An introduction to different strategy how to solve the fiber capacity crunch
---
## How to solve the AI fiber capacity crunch 

The rapid growth of AI-driven applications and the proliferation of massive datacenters have led to an unprecedented demand for optical network bandwidth, with requirements now reaching petabits per second. Traditional WDM (Wavelength Division Multiplexing) systems, which primarily utilize the C+L-band with capacities of around 50 Tbit/s, are no longer sufficient to meet this surge. To address this challenge, the industry is exploring several transformative innovations to sustain the explosive growth of data traffic and ensure the scalability of global optical networks.
 
These include: **Multi-rail transmission, C+L+S band transmission, Hollow Core/Multi Core Fiber** 

![Picture Blog capacity Fiber.jpg]({{site.baseurl}}/images/Picture Blog capacity Fiber.jpg)

## Parallel Transmission on More Fiber Pairs (Multi-Rail)
The sheer volume of data traffic has propelled us to a point where the capacity between interconnected data centers can no longer be quantified by the number of wavelengths. Instead, it is increasingly measured by the quantity of full-capacity fiber pairs needed to establish these critical links. 
We are now confronting scenarios demanding the deployment of ten, fifty, or even more than one hundred parallel fiber pairs, each fully equipped to operate across the entire C-band or the expanded C+L band. Consequently, there is an urgent imperative to efficiently distribute an immense amount of traffic, often reaching Exabyte scales, across these numerous parallel fiber pairs. This strategic distribution is vital for significantly reducing the overall power consumption and mitigating the substantial costs associated with scaling these vast network infrastructures.
Multi-Rail transmission achieves highly efficient bandwidth scaling by intelligently combining multiple independent optical pathways, aptly termed "rails," into a single, cohesive, and compact unit. Resource pooling, is achieved by implementing highly efficient pump lasers and dynamic multi-path gain equalization optics to optimize performance across all active rails.
Power Optimization for Multi-Rail transmission
Meticulous design optimization across rails is essential to effectively manage the demanding power requirements, particularly at inline amplifier (ILA) huts. These remote locations, where optical amplification is required for maintaining signal integrity over long distances, frequently face severe limitations in the electrical power they can support due to diverse factors such as remote accessibility, grid availability, or environmental constraints. A key enabling factor for the successful realization of such advanced WDM systems lies in specific developments within Integrated photonics is a key enabler for multi-rail transmission as it allows for the consolidation of multiple complex photonic functions onto a single, miniature chip, delivering substantial benefits in terms of real estate reduction, power efficiency, and overall cost optimization. This integration is crucial for making these high-capacity, multi-rail systems economically and operationally viable.


## Use More Transmission Bands on the Same Fiber
Traditionally, optical networks have utilized the C and L bands for data transmission. However, to increase capacity without the need for laying additional fiber, extending transmission into other bands such as the S-band is a promising solution. The availability of specialized optical amplifiers make the S-band a viable option for long-haul transmission.
- C-band (Conventional Band): Ranging from 1530 nm to 1565 nm, the C-band is the most widely adopted wavelength range in optical fiber communication. Its prominence stems from the fact that optical fiber exhibits the lowest attenuation (signal loss) in this band, making it ideal for long-distance transmission systems. The C-band is extensively used in metropolitan, long-haul, ultra-long-haul, and submarine optical communication networks, often integrated with Dense Wavelength Division Multiplexing (DWDM) and Erbium-Doped Fiber Amplifiers (EDFAs) for signal amplification. The advent of DWDM greatly expanded the utilization of the C-band.
- L-band (Long-wavelength Band): Spanning from 1565 nm to 1625 nm, the L-band is the second lowest-loss wavelength band. It is primarily employed when the capacity of the C-band alone becomes insufficient to meet bandwidth demands. The widespread availability of EDFAs has facilitated the expansion of DWDM systems into the L-band, significantly boosting the capacity of both terrestrial and submarine optical networks. By combining the C-band and L-band, the theoretical capacity of a single optical fiber can be substantially increased, potentially doubling it (e.g., from 25 Tbps to over 50 Tbps). Modern DWDM systems can support numerous channels within the combined C+L band.
- S-band (Short-wavelength Band): Covering wavelengths from 1460 nm to 1530 nm, the S-band exhibits lower loss compared to the O-band. While it has been widely used in Passive Optical Network (PON) systems, the S-band is increasingly being considered for DWDM applications to further enhance transmission capacity. Its low optical fiber loss (around 0.20-0.25 dB/km) and the availability of specialized optical amplifiers (like Thulium-Doped Fiber Amplifiers, Bismuth-Doped Fiber Amplifiers, or Lumped Raman Amplifiers) make it technically feasible for long-haul DWDM transmission, complementing the C and L bands.


The strategic utilization of the C, L, and S bands in WDM technology allows for a substantial increase in the data-carrying capacity of existing fiber optic infrastructure. This multi-band approach is crucial for supporting the escalating bandwidth requirements of modern communication networks, including the evolution of all-optical networks and the demands of high-bandwidth applications like data center interconnects for AI.
By leveraging these additional spectral bands on the same fiber infrastructure, network operators can significantly boost the data throughput. This approach maximizes the utilization of existing fiber assets, delays costly fiber deployment projects, and supports the growing bandwidth demands driven by modern applications and services.

## Hollow Core Fiber (HCF) and Multi Core Fiber (MCF)

### Hollow Core Fiber (HCF)
HCF represents a breakthrough in fiber technology by offering substantially increased bandwidth and lower latency compared to conventional Single Mode Fiber (SMF). Its unique structure allows for higher launch power, which can translate into longer reach and better signal quality. Hollow Core Fiber (HCF) technology distinguishes itself from traditional optical fibers by guiding light through a hollow central core, typically filled with air, inert gas, or a vacuum, rather than a solid glass or plastic core.
Unlike conventional fibers that rely on total internal reflection within a higher refractive index glass core, HCF employs a specially designed cladding structure to confine and transmit light within its hollow interior. This cladding often features a microstructural design, such as a photonic bandgap (PBG) crystal or anti-resonant structures, which are typically composed of a series of tiny air holes arranged periodically. This design prevents light of specific frequencies from escaping the core, effectively "bouncing" it back into the air-filled channel. This means that over 99% of the optical power propagates through air, minimizing interaction with solid materials.
Key Advantages:
HCF offers several significant benefits over traditional glass-core fibers:
- Lower Latency: Light travels approximately 30% faster in air or vacuum than in glass, leading to significantly reduced signal transmission delay. This makes HCF ideal for latency-sensitive applications like high-frequency trading and data center interconnects for AI applications.
- Lower Nonlinear Effects: The interaction between light and the transmission medium is significantly weakened in air, reducing nonlinear effects by 3 to 4 orders of magnitude compared to conventional glass fibers. This allows for higher input optical power and extended transmission distances.
- High-Power Laser Transmission: HCF is less susceptible to damage from high-power lasers (kilowatt-level) because over 99% of the optical power is transmitted through air, minimizing material absorption and increasing the laser damage threshold.
- Wider Bandwidth: The air core can support a broader range of optical wavelengths, potentially increasing bandwidth capabilities.
- Environmental Resilience: HCF can be more resistant to temperature fluctuations, radiation, and electromagnetic interference, making it suitable for extreme environments.
- Reduced Signal Loss (Attenuation): Because light primarily travels through air, which has much lower absorption and scattering losses than glass, HCF can achieve ultra-low attenuation, allowing for longer transmission distances without the need for frequent signal repeaters

The ongoing development of a common coherent Digital Signal Processing (DSP) platform that supports both SMF and HCF is crucial for seamless integration and adoption. However, widespread deployment of HCF depends on achieving manufacturing scale and ecosystem maturity to ensure cost-effectiveness and reliability.

### Multi-Core Fiber (MCF)
MCF technology involves multiple cores within a single fiber, which, when combined with Multi-Core Erbium-Doped Fiber Amplifiers (MC-EDFAs), can yield significant power and cost savings.
The core principle behind MCF is Spatial Division Multiplexing (SDM), where each independent core acts as a separate channel for data transmission. This enables a massive increase in per-fiber bandwidth without requiring more physical cables. For instance, a 4-core fiber can carry four times the channels of a single-core fiber of the same size
Key Advantages:
Key advantages of Multi-core fiber include:
- Massive Capacity Boost: It allows multiple light signals to run in parallel, dramatically increasing bandwidth.

- Space and Density Savings: By packing more cores into one fiber, MCF reduces the need for numerous individual fibers, saving space in conduits and data centers.

- Enhanced Network Efficiency: It contributes to more efficient network architectures due to its high data capacity and space-saving attributes.

- Improved Crosstalk Performance: Optimized core spacing and design help minimize interference between cores.

Despite these advantages, challenges remain, including higher manufacturing costs, operational complexities such as polarity management and FIFO (First In, First Out) design, and the necessity for dedicated Multiple-Input Multiple-Output (MIMO) DSP processing. These factors currently limit MCF's adoption primarily to niche scenarios, such as environments with limited conduit space where fiber count must be minimized.



These strategies collectively address the increasing capacity demands in optical networks by optimizing existing infrastructure, adopting innovative fiber technologies, and improving system design for power and cost efficiency. There are a lot of reference about those new technologies. You can learn more reading following articles:

Multi-Core Fibre: [High-capacity transmission using high-density multicore fiber](https://ieeexplore.ieee.org/abstract/document/7936906)

Hollow-Core Fibre: [Recent Progress in Low-Loss Hollow-Core Anti-Resonant Fibers and Their Applications](https://ieeexplore.ieee.org/abstract/document/8919973)

S-Band transmission: [Long-Haul >100-Tb/s Transmission Over >1000 km With High-Symbol-Rate Triple-Band WDM Signals](https://ieeexplore.ieee.org/document/10683993)
