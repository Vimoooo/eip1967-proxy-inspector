import os
from web3 import Web3

RPC_URL = os.getenv("ETH_RPC_URL", "https://ethereum-rpc.publicnode.com")

w3 = Web3(Web3.HTTPProvider(RPC_URL, request_kwargs={"timeout": 15}))

SLOTS = {
    "implementation": "eip1967.proxy.implementation",
    "admin": "eip1967.proxy.admin",
    "beacon": "eip1967.proxy.beacon",
}


def slot_key(name: str) -> str:
    return hex(int.from_bytes(Web3.keccak(text=name), "big") - 1)


def read_address_slot(contract: str, slot: str):
    raw = w3.eth.get_storage_at(contract, slot)

    if int.from_bytes(raw, "big") == 0:
        return None

    address = Web3.to_checksum_address(raw[-20:].hex())

    if address == "0x" + "0" * 40:
        return None

    return address


def inspect_proxy(address: str):
    address = Web3.to_checksum_address(address)
    code = w3.eth.get_code(address)

    if not code:
        raise ValueError("The address has no deployed contract code.")

    result = {
        "proxy": address,
        "implementation": read_address_slot(
            address, slot_key(SLOTS["implementation"])
        ),
        "admin": read_address_slot(
            address, slot_key(SLOTS["admin"])
        ),
        "beacon": read_address_slot(
            address, slot_key(SLOTS["beacon"])
        ),
    }

    if result["beacon"]:
        beacon_abi = [{
            "type": "function",
            "name": "implementation",
            "stateMutability": "view",
            "inputs": [],
            "outputs": [{"type": "address"}],
        }]

        try:
            beacon = w3.eth.contract(
                address=result["beacon"],
                abi=beacon_abi
            )
            result["beacon_implementation"] = (
                beacon.functions.implementation().call()
            )
        except Exception as error:
            result["beacon_implementation"] = (
                f"Could not read: {error}"
            )

    if result["implementation"]:
        result["proxy_type"] = "EIP-1967 implementation proxy"
    elif result["beacon"]:
        result["proxy_type"] = "EIP-1967 beacon proxy"
    else:
        result["proxy_type"] = "No standard EIP-1967 slots detected"

    return result


if __name__ == "__main__":
    target = input("Contract address: ").strip()

    try:
        report = inspect_proxy(target)

        print("\n--- Proxy Inspection Report ---")
        for key, value in report.items():
            print(f"{key}: {value}")

    except Exception as exc:
        print(f"Inspection failed: {exc}")
