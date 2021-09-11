# Fluence Character Counter

This is a Walk-Through on how to create a character count application using Fluence.

### Character Count Extension

The task is to extend the simple hello world example to add a character count to sent messages.

As you will have learned from the [Quick Start](https://doc.fluence.dev/docs/quick-start) tutorial, Fluence provides a Docker container pre-configured with example applications. We will be extending code in the examples/quickstart folder.


### Extending The Service

The rust code and configuration files for the service that will be deployed resides in the [quickstart/2-hosted-services](https://github.com/amrosaeed/fluence/tree/main/Fluence-Service/quickstart/2-hosted-services) folder. We modify the code in [src/main.rs](https://github.com/amrosaeed/fluence/blob/main/Fluence-Service/quickstart/2-hosted-services/src/main.rs) to add a character count to the message reply.

```rust
#[marine]
pub struct CharCount {
    pub msg: String,
    pub reply: String,
}

#[marine]
pub fn char_count(message: String) -> CharCount {
    let num_chars = message.chars().count();
    let _msg;
    let _reply;

    if num_chars < 1 {
        _msg = format!("Your message was empty");
        _reply = format!("Your message has 0 characters");
    } else {
        _msg = format!("Message: {}", message);
        _reply = format!("Your message {} has {} character(s)", message, num_chars);
    }

    CharCount {
        msg: _msg,
        reply: _reply
    }
}
```

Once this is done we build the service. This will create a WebAssembly file char_count.wasm in ./artifacts that will be deployed later.

```bash
cd quickstart
cd 2-hosted-services
./scripts/build.sh
```

* We update the tests in [src/main.rs](https://github.com/amrosaeed/fluence/blob/main/Fluence-Service/quickstart/2-hosted-services/src/main.rs) to work with the new service. 

* We enter a special character in the string to ensure that special characters are being counted correctly.

```rust
assert_eq!(actual.msg, "Message: SatoshiNakamoto");
assert_eq!(actual.reply, "Your message SatoshiNakamoto has 15 character(s)".to_string());
```

### Tests

A test is created in our `main.rs` file:

```rust
}

#[cfg(test)]
mod tests {
    use marine_rs_sdk_test::marine_test;

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn non_empty_string(char_count: marine_test_env::char_count::ModuleInterface) {
        let actual = char_count.char_count("SuperNode ☮".to_string());
        assert_eq!(actual.msg, "Message: SatoshiNakamoto");
        assert_eq!(actual.reply, "Your message SatoshiNakamoto has 15 character(s)".to_string());
    }

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn empty_string(char_count: marine_test_env::char_count::ModuleInterface) {
        let actual = char_count.char_count("".to_string());
        assert_eq!(actual.msg, "Your message was empty");
        assert_eq!(actual.reply, "Your message has 0 characters"); 
    }
}

```

Using the following command we test that our service is working as expected before deployment:

```bash
cargo +nightly test --release
```

### Deployment To Fluence

Once the service is built and the tests pass, we are ready to deploy the service to Fluence.

Services are deployed to specific peers that will handle our requests. We can gather a list of test peers using the command:

```bash
fldist env
```

We choose the top entry from the list for the deployment and use the node id with the fldist deployment command:

```bash
fldist --node-id 12D3KooWSD5PToNiLQwKDXsu8JSysCwUt8BVUJEqCHcDe7P5h45e \
       new_service \
       --ms artifacts/char_count.wasm:configs/char_count_cfg.json \
       --name char-count-br
```

Running this command returned the service id **1d818b13-37d8-4c27-8463-b6eee10f71e0** which we can use later to interact with the service.


### Updating The Aqua Code 

```TypeScript
import "@fluencelabs/aqua-lib/builtin.aqua"

const charCountNodePeerId ?= "12D3KooWSD5PToNiLQwKDXsu8JSysCwUt8BVUJEqCHcDe7P5h45e"
const charCountServiceId ?= "1d818b13-37d8-4c27-8463-b6eee10f71e0"

data CharCount:
  msg: string
  reply: string

-- The service runs on a Fluence node
service CharCount:
    char_count(from: PeerId) -> CharCount

-- The service runs inside browser
service CharCountPeer("CharCountPeer"):
    char_count(message: string) -> string

func countChars(messageToSend: string, targetPeerId: PeerId, targetRelayPeerId: PeerId) -> string:
    -- execute computation on a Peer in the network
    on charCountNodePeerId:
        CharCount charCountServiceId
        comp <- CharCount.char_count(messageToSend)

    -- send the result to target browser in the background
    co on targetPeerId via targetRelayPeerId:
        res <- CharCountPeer.char_count(messageToSend)

    -- send the result to the initiator
    <- comp.reply
```

### Compile aqua file
```
npm run compile-aqua
```

### Update aqua whith the new service (Update App.tsx)

In order to use the new service and display de char count you should update this file as is done in `3-browser-to-service/src/App.tsx

* We add new constant in function App() with two arguments messageToSend & setMessageToSend and use useState that is a hook helps us to manage a state string in function App().

```TypeScript
    const [messageToSend, setMessageToSend] = useState<string>(""); 
```
* By also adding Dialog component to App.tsx we allow adding more interactive feature to the application. (PLZ. Care for Indentation)

```TypeScript
 <input
className="input"
type="text"
placeholder="Type your message..."
onChange={(e) => setMessageToSend(e.target.value)}
value={messageToSend}
/> 
```
We also add a message box so that we can send our custom messages so that we can say more than a simple hello.

This requires an important change to pass the message (messageToSend) to the countChars method we defined earlier in the Aqua interface:

```TypeScript
  const messageBtnOnClick = async () => {
    if (client === null) {
      return;
    }
    // Using aqua is as easy as calling a javascript funсtion
    const res = await countChars(client!, messageToSend, peerIdInput, relayPeerIdInput);
    setMessage(res);
  };
```

### Running The Updated Application

The application is started by running:

```
npm start
```
Which will open a new browser tab at `http://localhost:3000` . Following the instructions, we connect to any one of the displayed relay ids, open another browser tab also at  `http://localhost:3000`, select a relay and copy and  paste the client peer id and relay id into corresponding fields in the first tab and press the `say hello` button.

1. connect to any one of the displayed relay ids:

![Pick-Relay](https://user-images.githubusercontent.com/82784007/132937525-2ccb2aa8-aae0-4634-ad08-e254b85d6f64.png)

2. open another browser tab also at  `http://localhost:3000`

![Duplicate-Tab](https://user-images.githubusercontent.com/82784007/132937569-dfbe7323-c473-4b10-b073-2cd94c2903f9.png)

3. select a relay and copy and  paste the client peer id and relay id into corresponding fields in the first tab and say somthing you think BITCOIN will last forever in the message fieldand press the `send message` button. :

![iLoveBITCOIN](https://user-images.githubusercontent.com/82784007/132937649-9687df43-2b19-4853-b16d-2ad12b7eec66.png)


4. We get back out message with the characters counted! and also our peer gets our message. :

![FluenceInAction](https://user-images.githubusercontent.com/82784007/132937687-19a566a8-9532-4485-9a04-7229bb3a8eed.png)


