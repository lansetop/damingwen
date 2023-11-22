from web3 import Web3, HTTPProvider, Account
from web3.middleware import geth_poa_middleware

private_key = "填写你的钱包私钥"
recipient_address = "填写你的接受钱包地址"
data = 'data:,{"p":"prc-20","op":"mint","tick":"pols","amt":"100000000"}'
rpc_map = {
    'mainnet': '填写你的rpc链接',
}


def get_transaction_eip1559(rpc_url, text_data, priority_fee):
    web3 = Web3(HTTPProvider(rpc_url))
    web3.middleware_onion.inject(geth_poa_middleware, layer=0)  # Inject POA middleware
    # Check if connected to Ethereum network
    if not web3.isConnected():
        raise Exception("Not connected to Ethereum network")
    # Set up the sender's account
    sender_account = Account.from_key(private_key)
    sender_address = sender_account.address

    # Transaction details
    value = web3.toWei(0, 'ether')
    # Get the nonce for the sender's account
    nonce = web3.eth.getTransactionCount(sender_address, 'pending')
    base_fee = web3.eth.get_block('latest')['baseFeePerGas']

    # Transaction details
    max_priority_fee_per_gas = web3.toWei(priority_fee, 'gwei')  # Tip for the miner, adjust as needed
    max_fee_per_gas = base_fee + max_priority_fee_per_gas
    # EIP-1559 transaction parameters

    # Convert data to hex and add as data to the transaction
    data_hex = web3.toHex(text=text_data)

    # Estimate gas limit for the transaction
    gas_estimate = web3.eth.estimate_gas({
        'to': recipient_address,
        'value': value,
        'from': sender_address,
        'data': data_hex
    })

    # Create the transaction dictionary
    transaction = {
        'type': '0x2',  # Indicates an EIP-1559 transaction
        'chainId': web3.eth.chain_id,
        'nonce': nonce,
        'maxPriorityFeePerGas': max_priority_fee_per_gas,
        'maxFeePerGas': max_fee_per_gas,
        'gas': gas_estimate,
        'to': recipient_address,
        'value': value,
        'data': data_hex
    }
    print(f"Transaction: {transaction}")
    return transaction, web3, private_key, nonce


def signed_send(transaction, web3, private_key, is_wait=True):
    # Sign the transaction
    signed_tx = web3.eth.account.sign_transaction(transaction, private_key)
    # Send the transaction
    tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction)
    # Get the transaction hash
    print(f"Transaction hash: {tx_hash.hex()}")
    # Wait for the transaction receipt (optional)
    if is_wait:
        tx_receipt = web3.eth.waitForTransactionReceipt(tx_hash)
        print(f"Transaction receipt: {tx_receipt}")
        print(f"Transaction status: {tx_receipt['status']}")


def send_transaction(number, rpc, test_data, is_wait=True, priority_fee=400):
    transaction, web3, private_key, nonce = get_transaction_eip1559(rpc, test_data, priority_fee)
    for i in range(number):
        transaction.update({'nonce': nonce})
        signed_send(transaction, web3, private_key, is_wait)
        nonce = nonce + 1


if __name__ == '__main__':
    send_transaction(1, rpc_map.get("mainnet"), data, is_wait=True, priority_fee=100)