# PCIe fabrics and SM120 research resources

Added October 9, 2026 after checking the current local research notes. This adds the Local Inference Lab wiki and the broader workstation GPU switch/backplane topic to our reading list. Rocinante was unloaded for a physical shutdown; this intake ran on the Mac and made no workstation configuration changes.

## Sources to revisit

| Source | Why it belongs here | Inspected revision |
|---|---|---|
| [Local Inference Lab: rtx6kpro](https://github.com/local-inference-lab/rtx6kpro) | Blackwell workstation inference, PCIe topology, serving recipes, kernel work and benchmark methodology | `efcd53414f2937a8c2f6551eb6f589850b071705` |
| [PCIe bandwidth measurements](https://github.com/local-inference-lab/rtx6kpro/blob/efcd53414f2937a8c2f6551eb6f589850b071705/hardware/pcie-bandwidth.md) | Pairwise copies, remote-memory kernel access and collectives need separate checks | Blob `9544778af859dce9fa48c5a29118a75335ea41d3` |
| [Topology guide](https://github.com/local-inference-lab/rtx6kpro/blob/efcd53414f2937a8c2f6551eb6f589850b071705/hardware/topology.md) | Direct attachment, shared host uplinks and routes between switch groups | Blob `f59e89af940d13b849c713a3572f6aac60e0f20f` |
| [James O'Beirne: local-llm](https://github.com/jamesob/local-llm/blob/4172e40db404ecb6e3e97a0fae397364dfda9c6f/README.md) | A concrete DDR4 EPYC build using four 96 GB cards and a c-payne switch | Commit `4172e40db404ecb6e3e97a0fae397364dfda9c6f`; README blob `064c561fb25c73c265191e4e1eb6ab61ef699cce` |

These are external build reports. Their measurements are not Rocinante results.

## What the reports establish

The wiki's bandwidth page reports approximately 53–56 GB/s **unidirectional** transfers on several Gen5 x16 systems. Its eight-GPU, three-switch row reports a 191.8 GB/s dense-interconnect result. That aggregate benchmark is a different quantity from a pairwise link rate or NCCL bus bandwidth. The page also distinguishes copy-engine transfers from kernels accessing peer memory directly: a good copy result alone does not validate the latter path. [Source](https://github.com/local-inference-lab/rtx6kpro/blob/efcd53414f2937a8c2f6551eb6f589850b071705/hardware/pcie-bandwidth.md)

The topology guide describes local switch routing that avoids the host path, while independent switch groups can still communicate through CPU root ports. Their shared upstream link also constrains traffic to host RAM. The tradeoff therefore depends on the workload's communication pattern. [Source](https://github.com/local-inference-lab/rtx6kpro/blob/efcd53414f2937a8c2f6551eb6f589850b071705/hardware/topology.md)

O'Beirne's build uses an EPYC 7313P, ROMED8-2T, four RTX PRO 6000s and a Microchip PM40100 Gen4 switch. Two SlimSAS x8 cables provide one x16 upstream connection through a redriver adapter. His recorded switch sub-BOM totals about €1,220; this is a dated build cost. He reports 27.5 GB/s unidirectional and 50.4 GB/s bidirectional P2P, with 0.37–0.45 microsecond latency. His notes document link training, cable choice, BIOS settings and ACS/IOMMU changes used on that machine. [Source](https://github.com/jamesob/local-llm/blob/4172e40db404ecb6e3e97a0fae397364dfda9c6f/README.md)

## How this applies to Rocinante

Our inference: an active switch could solve physical fan-out and provide a better route between GPUs if their transfers are on the critical path. An ordinary riser only relocates a card; it does not create this switching fabric. Neither arrangement combines VRAM into one automatically shared pool: model placement and the runtime still determine how each GPU participates.

For the existing PRO 5000 plus 4070 Ti Super, first compare direct attachment and Strata's no-P2P peer expert path. Record the actual link generation/width under load and peer support in each direction. Then measure both copies and the particular remote-expert or collective operations the engine uses. Two unequal cards are a different problem from four or eight identical PRO 6000s.

Strata also transfers data from system RAM. Our next comparison must account for that traffic alongside GPU-to-GPU traffic. A switch that improves peer communication can introduce contention on its host uplink; larger aggregate peer bandwidth alone does not predict a faster completed task.

Retain the model/quant, speculation settings and 32K–100K task protocol while comparing placement. Record cold prefill, warm follow-ups, output completion, task correctness, transfer waits and total elapsed time. PCIe bandwidth is a diagnostic, not the final outcome.

## Additional leads from the user's supplied overview

The following remain queued for primary-source verification:

- Killy / `@net_termina`: Broadcom PEX89104 versus multi-switch c-payne trees, including routes between switch chips.
- Brandon Music: host-to-Gen5-switch cabling using retimers and MCIO.
- Japanese four-PRO-6000/older-EPYC build coverage: exact switch generation, measured transfers and cost.
- HighPoint Rocket Gen5 MCIO switch adapters and bridge cards: exact models, supported GPU/P2P paths and host uplink width.
- Zach's MWIN backplane: exact board, routing diagram and reproducible pairwise/collective measurements.

Names and claims in this last section came from the supplied overview; they have not been independently confirmed in this intake. External ACS, IOMMU and driver recipes are configuration-specific research material, not changes made to Rocinante.
