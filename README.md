# base221zq
0x81b057dD6B2bAA9584aE5f129bAd27415A18D1E7
import time
from collections import defaultdict, deque
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
CYCLE_THRESHOLD = 5

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting repeating trader behavior...\n")

    last_block = w3.eth.block_number

    # wallet -> token -> recent directions
    history = defaultdict(
        lambda: defaultdict(lambda: deque(maxlen=10))
    )

    scores = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )


                    if sender == ZERO:
                        continue


                    # outgoing
                    history[sender][token].append("OUT")

                    # incoming
                    if receiver != ZERO:
                        history[receiver][token].append("IN")


                for wallet, tokens in history.items():

                    for token, actions in tokens.items():

                        if len(actions) < 6:
                            continue

                        pattern = list(actions)

                        flips = 0

                        for i in range(1, len(pattern)):

                            if pattern[i] != pattern[i-1]:
                                flips += 1


                        if flips >= CYCLE_THRESHOLD:
                            scores[wallet] += 1


                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, score in scores.items():

                    if score >= 3:

                        print("🔁 Repeating Trader")
                        print("Wallet:", wallet)
                        print("Cycle score:", score)
                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
