Theory of Operation: DeMoDical Unidirectional Transport
Document ID: DMD-TOO-100G-01
Date: December 23, 2025
Author: DeMoD LLC Engineering
This document outlines the theoretical basis and operational logic of the DeMoDical high-speed optical bus. Unlike standard bidirectional protocols (TCP/IP, PCIe, InfiniBand) which rely on handshakes and re-transmission requests (ARQ) to guarantee integrity, the DeMoDical operates as a simplex (one-way) "firehose."
To ensure data integrity and link stability without a return channel, the system relies on Forward Error Correction (FEC), Deterministic Rate Control, and Blind Signal Locking.
1. Physical Layer (PHY): The "Blind Lock" Strategy
In a standard 100G Ethernet link, the Transmitter (TX) and Receiver (RX) negotiate parameters (Auto-Negotiation) and train their equalizers (Link Training). The DeMoDical removes this negotiation.
 * Plesiochronous Clocking: The TX and RX operate on separate, high-precision reference clocks (typically 156.25 MHz or 322.26 MHz). Because they are not phase-locked, there will be a parts-per-million (ppm) frequency drift between them.
 * Continuous Clock Data Recovery (CDR): The Receiver's FPGA transceivers (GTY/GTH) are configured in "Lock-to-Data" mode. The TX sends a continuous stream of DC-balanced symbols. The RX CDR circuit extracts the clock directly from the data transitions, allowing it to track the TX clock phase without a reference cable.
 * Signal Integrity: We utilize 4 lanes of 25.78125 Gbps (NRZ or PAM4). Because there is no backchannel to request "tuning" of the signal pre-emphasis, the TX uses a fixed, robust equalization profile optimized for the specific cable length (AOC) defined in the spec (5–30m).
2. Data Link Layer: Encoding & Scrambling
Raw data cannot be sent directly over optical fiber because long strings of zeros or ones (CID - Consecutive Identical Digits) would cause the Receiver's CDR to lose lock.
 * 64b/66b Encoding: The DeMoDical maps 64 bits of payload data into 66-bit "blocks." The first 2 bits are a "sync header" (01 for data, 10 for control). This guarantees a transition at least once every 66 bits, providing the "heartbeat" the RX needs to stay synchronized.
 * Scrambling: A pseudo-random bit sequence (PRBS) scrambler is applied to the payload before encoding. This "whitens" the spectrum, ensuring that the energy is spread evenly across the frequency band and preventing electromagnetic interference (EMI) spikes.
3. Error Correction Strategy (FEC)
The critical challenge of one-way communication is that data loss is usually permanent. We cannot ask the sender to "please repeat that."
 * No ARQ (Automatic Repeat Request): Traditional "Reliable" protocols acknowledge every packet. We assume the link is lossy.
 * Reed-Solomon Forward Error Correction (RS-FEC): We implement a Reed-Solomon (528, 514) or (544, 514) block code.
   * Theory: The encoder adds redundant parity symbols to the data stream.
   * Operation: If a bit flips during transmission (due to cosmic rays, heat, or jitter), the RX mathematics engine uses the parity symbols to mathematically reconstruct the original bit.
   * Capacity: This allows the system to correct burst errors of up to 7–15 consecutive symbols without losing any data. If the error rate exceeds this threshold, the packet is flagged as "corrupt" and dropped, preserving the integrity of the stream (fail-secure).
4. Flow Control & Rate Adaptation
The "Sanity" Problem: If the TX sends data at exactly 100.000 Gbps, and the RX processes it at 99.999 Gbps (due to clock variance), the RX buffer will eventually overflow and data will be lost.
 * TX Rate Limiting (The "Gap" Method): The Transmitter is hard-coded to insert "IDLE" or "FILLER" patterns into the stream at fixed intervals (e.g., 98% payload, 2% idle).
 * RX Elastic Buffer: The Receiver uses a FIFO (First-In-First-Out) elastic buffer.
   * It writes data using the recovered clock (from the fiber).
   * It reads data using the local system clock.
 * Sanity Mechanism: When the RX sees an "IDLE" pattern, it discards it. This allows the RX buffer to "drain" slightly, compensating for clock drift and ensuring it never overflows.
5. Summary of Signal Chain
 * Input: User Data (PCIe/AXI Stream)
 * FEC Encoder: Add parity bits.
 * Scrambler: Randomize bits for spectral density.
 * Framer: Apply 64b/66b headers.
 * Serializer (SerDes): Convert parallel data to 25G serial stream.
 * Optical TX: Laser modulation.
   --- Optical Barrier (The Diode) ---
 * Optical RX: Photodiode detection.
 * CDR: Lock onto clock & align bits.
 * Deskew: Align the 4 parallel lanes.
 * FEC Decoder: Fix bit errors.
 * Output: Validated Data Stream.
Next Step
Would you like me to generate a Python/migen (LiteX) or Verilog snippet demonstrating the 64b/66b scrambling logic or the Elastic Buffer mechanism for this architecture?

