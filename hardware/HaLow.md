
# Reticulum network device 

<img width="789" height="783" alt="image" src="https://github.com/user-attachments/assets/1be11858-125e-4a4a-90f3-59f5f60e9f8a" />


- Sodium ION MPPT charging for <0C [Exists, needs testing](https://easyeda.com/editor#id=0b6352d4c879468b8ee60e0a43bd0ce0) acts as main plug and play carrier for below comoponents. todo: Find [large capacity Sodium](https://www.aliexpress.com/item/1005010152325470.html) rated for <0C charging
- Low power open hardware SBC for OpenWRT (Possibly [Avaota V861](https://www.cnx-software.com/2026/01/04/allwinner-v861-dual-core-64-bit-risc-v-ai-camera-sip-features-128mb-ddr3l-4k-h-265-h-264-video-encoder/) chips are [cheap](https://www.alibaba.com/product-detail/Allwinner-V821-ic-chip-SIP-wifi_1601384222436.html) [6.xx kernel WRT 21.xx](https://x.com/GLGH_/status/2005219500774560217)
- Two HaLow Network devices in XIAO format for later upgrading [XIAO HaLow](https://oshwlab.com/robcarey/3-0067)
- 433Mhz LoRa interface (On 433Mhz in EU to minimise interference using) [RAK4631](https://store.rakwireless.com/products/rak4631-lpwan-node?utm_source=website_partner&utm_medium=referral&utm_campaign=meshtastic_rak_collab&variant=41425862328518)

![alt text](https://www.cnx-software.com/wp-content/uploads/2026/01/Allwinner-V861-dual-camera-board-720x543.webp "Avaota SBC")




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

