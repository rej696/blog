# Networking and Comms Notes

## Network Guides
- Beejs guide to Network Concepts (Python): https://beej.us/guide/bgnet0/html/split/
- Beejs guide to Network Programming (Sockets with C): https://beej.us/guide/bgnet/html/split/

## Error detection and Correction (EDAC)
Notes summarised from https://en.wikipedia.org/wiki/Error_detection_and_correction

### Error Correction
- Automatic repeat request (ARQ):
    - use error detection to trigger retransmission
    - e.g. nacking the receipt of a packet (see flow control)
- Forward Error Correction (FEC):
    - Add redundant data such as an error correcting code
    - Allow some number of errors to be recovered without retransmission
    - Convolutional Codes: bit-by-bit
        - viterbi algorithm for decoding
    - Block codes: block-by-block
        - hamming codes
        - reed solomon
        - turbo codes
        - low density parity check codes (LDPC)
- Hybrid:
    - transmit with FEC, but request retransmit if unable to decode with FEC
    - transmit without FEC, but rerequest FEC information if error detected to try to correct message

### Error Detection
- Repetition codes:
    - blocks of bits are repeated some predetermined number of times, e.g. 3 times
    - errors are detected when the blocks are different from one another
- Parity Bit:
    - parity bit flags if the number of on bits (i.e 1) is even or odd
    - can detect single or an odd number of bit flips
    - transverse redundancy checks are parity bits attached to a word (i.e. 8N1 in serial)
    - longitudinal redundancy checks are bits added to the end of a stream of words
- Checksum:
    - modular aritmetic sum of the words in a message of a fixed word length (i.e. bytes)
    - `for byte in message: csum = csum + byte % 256`
    - various other checksum algorithms and schemes
- Cyclic Redundancy Check (CRC):
    - non-secure hash function of a message for detecting changes in data
    - some generator polynomial is specified, which is used as the divisor in
      polynomial long division using input data as the dividend. the remainder
      is the result which is used to detect errors
- List of hash functions: https://en.wikipedia.org/wiki/List_of_hash_functions

## Flow Control:
- Managing the rate of data transmission between two nodes
- stop and wait:
    - send data, receive ack, etc.
- sliding window:
    - receiver gives trasmitter permission to transmit until a window is full
    - frames have sequence numbers
    - multiple frames can be transmitted before the all the acks are received
    - window is a number of frames less than a receive buffer size
    - reliable in order delivery of packets
    - e.g. TCP
- flow control can be performed either with:
    - control signal lines
    - reserving characters/symbols to signal flow start/stop

### Hardware flow control
- RS-232 has a pair of control lines which are referred to as "hardware flow control"
    - RTS (request to send) and CTS (clear to send)

