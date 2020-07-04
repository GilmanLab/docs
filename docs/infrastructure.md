# Gilman Lab Infrastructure

This page contains (mostly) up to date information about the underlying physical
infrastructure which serves as the foundation for the Gilman Lab. The equipment
represented here is a culmination of about three years of running a homelab.
What's not shown here is the many evolutions that have happened as experience is
gained running different brands/types of equipment. The hope is that it can
serve as a reference to assist others who may be interested in breaking into the
homelab world.

## Design

The following are the primary design principles that went into building out the
infrastructure:

-   Silent - A reference point of 50db was chosen, meaning all equipment running
    at normal load should not produce more than 50db of noise.
-   Efficient - The only ongoing cost of a homelab is power consumption. Thus,
    where possible, more modern platforms which take advantage of recent
    advancements to provide more power with less TDP were chosen. For custom
    builds, highly efficient power supplies and auxiliary cooling were chosen.
-   Compact - The design size of the lab is 22u. Equipment was chosen based on
    size and the expectation was that it would all fit within an 22u rack with
    no overflow.
-   Expandable - Where possible, the lab should be relatively easy to grow over
    the years as needed. This means not relying on niche parts that are hard to
    come by and using identical components when scaling out.

## Architecture

### Compute

-   3 x
    [NUC8i7HVK](https://www.amazon.com/Intel-NUC-Performance-G-Kit-NUC8i7HVK/dp/B07BR5GK1V)
    -   1 x
        [Samsung 970 EVO SSD 1TB](https://www.amazon.com/gp/product/B07BN217QG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
        (Capacity)
    -   1 x
        [Samsung 970 EVO Plus SSD 250GB](https://www.amazon.com/gp/product/B07MG119KG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
        (Cache)
    -   2 x
        [Samsung 32GB DDR4 2666MHz RAM](https://www.amazon.com/gp/product/B07N124XDS/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
-   2 x Custom build
    -   1 x
        [iStarUSA D-400L-7](https://www.amazon.com/iStar-D-400L-7-Rackmount-Server-Chassis/dp/B005KQ653K)
    -   1 x
        [Supermicro X11SPM-TF](https://www.amazon.com/Supermicro-X11SPM-TF-Server-Motherboard-LGA-3647-1/dp/B0752ZVT8Q/)
    -   1 x
        [EVGA SuperNOVA 850 P2, 80+ PLATINUM 850W](https://www.amazon.com/dp/B010HWDOH6?tag=pcpapi-20&linkCode=ogi&th=1)
    -   1 x
        [Intel Xeon Silver 4214 12-Core](https://www.amazon.com/Intel-BX806954214-2-2GHz-FC-LGA14B-Retail/dp/B07R91QLTM)
    -   4 x
        [Samsung DDR4 2133MHzCL15 32GB](https://www.amazon.com/gp/product/B00URYU93C/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
    -   1 x
        [Samsung 970 EVO SSD 1TB](https://www.amazon.com/gp/product/B07BN217QG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
        (Capacity)
    -   1 x
        [Samsung 970 EVO Plus SSD 250GB](https://www.amazon.com/gp/product/B07MG119KG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
        (Cache)
    -   1 x
        [M.2 NVME to PCIe 3.0 x4 Adapter](https://www.amazon.com/YATENG-Controller-Expansion-Card-Support-Converter/dp/B07JJTVGZM)
    -   1 x
        [Noctua NH-U12S DX-3647](https://www.amazon.com/Noctua-NH-U12S-DX-3647-Premium-Quality/dp/B07DPSXNK2)
    -   3 x
        [Noctua NF-S12B redux-1200 PWM](https://www.amazon.com/Noctua-NF-S12B-redux-1200-PWM-Performance/dp/B00KF7PPY4)

### Network

-   [Netgate XG-7100](https://www.netgate.com/solutions/pfsense/xg-7100-1u.html)
-   [Mikrotik CRS317-1G-16S+RM](https://www.amazon.com/gp/product/B0747TC9DB/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
-   [Mikrotik CSS326-24G-2S+RM](https://www.amazon.com/gp/product/B0723DT6MN/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
-   Various SFP's from fs.com

### Storage

-   [Synology RS818+](https://www.amazon.com/gp/product/B078W5B2GL/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
    -   4 x
        [WD Red Pro 6TB](https://www.amazon.com/gp/product/B01CHP20MG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
    -   1 x
        [Intel Ethernet Converged X710-DA2](https://www.amazon.com/gp/product/B00NJ3ZC26/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)

### Power

-   [CyberPower PR1500LCDRT2U](https://www.amazon.com/gp/product/B001E06YC8/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)

### Rack & Misc

-   [Sysracks SRF 22.6.10 22u Enclosed Rack](https://sysracks.com/catalog/sysracks-srf/22u-39-depth-it-telecom-cabinet-srf-22-6-10/)
-   [Tripp Lite 24Port Shielded Blank Patch Panel](https://www.amazon.com/gp/product/B01LWYF3MA/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
-   [Monoprice SlimRun Cat6A](https://www.amazon.com/gp/product/B01BGV2P5E/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
-   [VELCRO One-Wrap Cable Ties](https://www.amazon.com/gp/product/B001E1Y5O6/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
