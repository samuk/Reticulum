
# Reticulum network device 

- Sodium ION MPPT charging for <0C
- HaLow Network device in Mikroe click format for later upgrading (Port this [XIAO one](https://oshwlab.com/robcarey/3-0067
))
- Low power open hardware SBC for OpenWRT (Possibly [Avaota V861](https://www.cnx-software.com/2026/01/04/allwinner-v861-dual-core-64-bit-risc-v-ai-camera-sip-features-128mb-ddr3l-4k-h-265-h-264-video-encoder) chips are [cheap](https://www.alibaba.com/product-detail/Allwinner-V821-ic-chip-SIP-wifi_1601384222436.html)
- LoRa interface (On 433Mhz in EU to minimise interference using XIAO footprint)



# Notes



MM8180 from Morse Micro is now at digikey for £18 There's an open hardware carrier for it in Xiao footprint and It's EU compliant. https://github.com/orgs/OpenMANET/discussions/9#discussioncomment-15278985


There is also a (expensive!) MM8108-EKH19, usb version docs My guess/hope would be that an open hardware version with £100 knocked off the price will be along shortly
https://www.mouser.co.uk/ProductDetail/Morse-Micro/MM8108-EKH19-01?qs=HMhDvBYWqvOSMbg%2FkiOXMw%3D%3D
https://www.morsemicro.com/resources/datasheets/modules/MM8108-MF15457_Data_Sheet.pdf

If I was to pair this with an open hardware TaishanPi or upcoming avaota board
https://www.cnx-software.com/2026/01/04/allwinner-v861-dual-core-64-bit-risc-v-ai-camera-sip-features-128mb-ddr3l-4k-h-265-h-264-video-encoder/
and put OpenWRT I could then put Reticulum on it

https://openkits.easyeda.com/project/detail/lctspi-rk3566-1g-0g?param=baseInfo&collection=ec3dd58694bf484ca6ca2e2cbd810a99
https://github.com/ophub/amlogic-s9xxx-openwrt
https://github.com/gretel/reticulum-openwrt

Would I then need to run batman-adv on OpenWRT to get a Halow mesh? Or would Reticulum sort out a mesh without needing batman-adv underneath it?
https://github.com/benkay86/openwrt-batman-tutorial

It seems like Morse Micro have a somewhat hacky mesh implimentation already: MorseMicro/linux@bbcf1ad
https://github.com/MorseMicro/linux/commit/bbcf1ad4ad009087432c74a066d7741e2f06b56e

