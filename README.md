# Open Frames Spec [Draft v0.0.3]

Since the launch of [Farcaster Frames](https://docs.farcaster.xyz/reference/frames/spec), we’ve seen a number of protocols work to add Frames support independently. The original Frames spec was designed for Farcaster and wasn’t set up to handle interactions from different types of applications with unique capabilities. Open Frames is a lightweight extension to the Frames spec to help coordinate the many new applications and protocols adopting Frames.

Open Frames is an initiative with the following goals:

1. Allow Frames developers to support interactions from different types of clients
2. Don’t screw up the best thing about Frames: how easy it is to build on

We expect the Open Frames specification to evolve, both through improvements to the Farcaster Specification and through the needs of Open Frames applications with different capabilities.

## Actors

### Client Application

Client applications are responsible for rendering Frames for end-users, handling interactions with buttons on Frames, and creating the POST payload to be sent to the Frames Server. Client applications must support one or more `clientProtocols`.

### Frames Server

A Frames Server is responsible for generating the initial Frames HTML, handling `POST` requests from Client Applications, and storing any state that might be required for the functioning of the Frame.

## Frames Metadata

To turn your web pages into Frames, you need to add basic metadata to your page.

### Required Properties

| Property | Description |
| --- | --- |
| `of:version`  | The version label of the Open Frames spec. Currently the only supported version is `vNext` |
| `of:accepts:$protocol_identifier` | The minimum client protocol version accepted for the given protocol identifier. For example `vNext` , or `1.5`. At least one `$protocol_identifier` must be specified. |
| `of:image` | An image which should have an aspect ratio of `1.91:1` or `1:1`.  |
| `og:image` | An image which should have an aspect ratio of `1.91:1`. Fallback for clients that do not support frames. |

### Optional properties

| Property | Description |
| --- | --- |
| `of:button:$idx` | 256 byte string containing the user-visible label for button at index `$idx`. Buttons are 1-indexed. Maximum 4 buttons per Frame. `$idx` values must be rendered in an unbroken sequence.   |
| `of:button:$idx:action` | Valid options are `post`, `post_redirect`, `mint`, `link`, and `tx`. Default: `post` |
| `of:button:$idx:target` | The target of the action. For `post` , `post_redirect`, and link action types the target is expected to be a URL starting with `http://` or `https://`. For the mint action type the target must be a [CAIP-10 URL](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md) |
| `of:button:$idx:post_url` | 256-byte string that defines a button-specific URL to send the POST payload to. If set, this overrides `of:post_url`
| `of:post_url` | The URL where the POST payload will be sent. Must be valid and start with `http://` or `https://` . Maximum 256 bytes. |
| `of:input:text` | If this property is present, a text field should be added to the Frame. The contents of this field will be shown to the user as a label on the text field. Maximum 32 bytes. |
| `of:image:aspect_ratio` | The aspect ratio of the image specified in the `of:image` field. Allowed values are `1.91:1` and `1:1`. Default: `1.91:1` |
| `of:image:alt` | Alt text associated with the image for accessibility |
| `of:state` | A state serialized to a string (for example via JSON.stringify()). Maximum 4096 bytes. Will be ignored if included on the initial frame |
| `of:context:$protocol_identifier` | Boolean value specifying whether requests sent to frame server must include context (untrustedData) corresponding to the client protocol. Default = `true` | 

## Button actions

### `post`

The `post` action sends a HTTP POST request to the frame or button `post_url`. This is the default button type.

The frame server receives a signed frame action payload in the POST body, which includes information about which button was clicked, text input, and the cast context. The frame server must respond with a `200 OK` and another frame.

## `post_redirect`

The `post_redirect` action sends an HTTP POST request to the frame or button `post_url`. You can use this action to redirect to a URL based on frame state or user input.

The frame server receives a signed frame action payload in the POST body. The frame server must respond with a `302 Found` and `Location` header that starts with `http://` or `https://`.

## `link`

The `link` action redirects the user to an external URL. You can use this action to redirect to a URL without sending a POST request to the frame server.

Clients do not make a request to the frame server for link actions. Instead, they redirect the user to the `target` URL.

## `mint`

The `mint` action allows the user to mint an NFT. Clients that support relaying or initiating onchain transactions may enhance the mint button by relaying a transaction or interacting with the user's wallet. Clients that do not fall back to linking to an external URL.

The target property must be a valid [CAIP-10](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md) address, plus an optional token ID appended with a `:`.

## `tx`

The `tx` action allows a frame to send a transaction request to the user's connected wallet. Unlike other action types, tx actions have multiple steps.

First, the client makes a POST request to the `target` URL to fetch data about the transaction. The frame server receives a signed frame action payload in the POST body, including the address of the connected wallet in the `address` field. The frame server must respond with a `200 OK` and a JSON response describing the transaction which satisfies the following type:


```ts
type TransactionTargetResponse {
  chainId: string;
  method: "eth_sendTransaction";
  params: EthSendTransactionParams;
}
```

### Ethereum Params

If the method is `eth_sendTransaction` and the chain is an Ethereum EVM chain, the param must be of type `EthSendTransactionParams`:

- `abi`: JSON ABI which **MUST** include encoded function type and **SHOULD** include potential error types. Can be empty.
- `to`: transaction recipient
- `value`: value to send with the transaction in wei (optional)
- `data`: transaction calldata (optional)

```ts
type EthSendTransactionParams {
  abi: Abi | [];
  to: `0x${string}`;
  value?: string;
  data?: `0x${string}`;
}
```

Example:

```json
{
  "chainId": "eip155:1",                                // The chain ID of the transaction
  "method": "eth_sendTransaction",                      // The method to call on the wallet
  "params": {
    "abi": [...],                                       // JSON ABI of the function selector and any errors
    "to": "0x0000000000000000000000000000000000000001", // The recipient of the transaction
    "data": "0x00",                                     // Transaction calldata
    "value": "123456789",                               // Value to send with the transaction
  },
};
```

The client then sends a transaction request to the user's connected wallet. The wallet should prompt the user to sign the transaction and broadcast it to the network. The client should then send a POST request to the `post_url` with a signed frame action payload including the transaction hash in the `transactionId` field to which the frame server should respond with a `200 OK` and another frame.

## Determining the `post_url`

When buttons with `post` and `post_redirect` actions are pressed, the client must send a POST request with the frame action payload to the URL determined by the following rules:

1. If the the button specifies a `target`, the POST request must be sent to the `target` URL.
2. If the button specifies a `post_url`, the POST request must be sent to the `post_url` URL if `target` is not present
3. If the button does not specify a `target` or `post_url`, the POST request must be sent to the URL specified in the `of:post_url` property of the Frame metadata.
4. If the Frame metadata does not specify a `post_url`, the POST request must be sent to the URL of the Frame.


## Farcaster Compatibility

The following properties are directly compatible with the following Farcaster properties as of Feb 21, 2024

| Open Frames Property | Farcaster Property |
| --- | --- |
| `of:image` | `fc:frame:image` |
| `og:image` | `og:image` |
| `of:button:$idx` | `fc:frame:button:index` |
| `of:button:$idx:action` | `fc:frame:button:$idx:action` |
| `of:button:$idx:target` | `fc:frame:button:$idx:target` |
| `of:button:$idx:post_url` | `fc:frame:button:$idx:post_url` |
| `of:input:text` | `fc:frame:input:text` |
| `of:image:aspect_ratio` | `fc:frame:image:aspect_ratio` |
| `of:accepts:farcaster` | `fc:frame` |
| `of:state` | `fc:frame:state` |

Frames Servers that wish to handle both Farcaster and Open Frames interactions are recommended to include a complete set of both `of:` and `fc:frame` meta tags. If a complete set of Open Frames tags are not present in the page, Client Applications may choose to fall back to equivalent `fc:frame` tags if the `of:accepts:$protocol_identifier` tag is present.

## The Client Protocol Identifier

`clientProtocol` is a string identifier for a client’s capabilities that can be used to negotiate compatibility between the Client Application and the Frame Server. Each Frame Server must advertise which `clientProtocol`s it is capable of receiving button clicks from through the `of:accepts:$protocol_identifier` meta tag included in each HTML response.

```html
<meta 
	property="of:accepts:xmtp"
	content="2024-02-01"
/>
<meta 
	property="of:accepts:lens"
	content="1.1"
/>
```

If a frame does not require authentication from any client protocol, the frame can include the tag `of:accepts:anonymous`. If the anonymous tag is included with other client protocol tags, then it is optional for a request to this Frame Server to include authentication for the other client protocol(s).

If a Frame Server declares that it is compatible with a client protocol, Client Applications capable of sending POST responses using that protocol should feel confident that the Frame will work as expected.

Client protocol strings must conform to the following format: `$PROTOCOL_IDENTIFIER@$PROTOCOL_VERSION`. Protocol versions are a string, and it is up to each protocol to decide how it should specify versions. For example, a Farcaster client today might define it’s `clientProtocol` as `farcaster@vNext`. 

The version specified in the `of:accepts:*` tag should be the earliest version of the client protocol with which the Frames Server is compatible.

Client applications must check the accepts tag for their protocol and ensure that they are capable of sending POST payloads in a format that the Frame Server can understand. If the
`of:accepts:$client_protocol` field is absent, client applications may choose to assume that the Frame Server only accepts requests using the Farcaster request format using the version specified in the `fc:frame` meta tag. If the client application does not support any of the listed client protocols, the client can choose to skip rendering the Frame entirely or show the Frame with the buttons disabled.

When sending a POST to the Frame Server, client applications must include the `clientProtocol` used to generate the payload, which will allow the Frame server to know what data is available and how to verify the `trustedData.messageBytes`.

## `POST` Payloads

When a user clicks a button on a Frame, the Frame developer receives a `POST` request with a payload containing both `untrustedData` and `trustedData`. The `trustedData.messageBytes`
can be verified by Frame developers so that they can provably know that the contents of that payload were signed by a given user. We propose a minimum schema for all `POST` payloads that can be implemented by any Open Frames `clientProtocol`.

```tsx
type FramesPost = {
  clientProtocol: string; // The client protcol used by the client to generate the payload
  untrustedData: {
    url: string; // The URL of the Frame that was clicked. May be different from the URL that the data was posted to.
    unixTimestamp: number; // Unix timestamp in milliseconds
    buttonIndex: number; // The button that was clicked
    inputText?: string; // Input text for the Frame's text input, if present. Undefined if no text input field is present
    state?: string; // State that was passed from the frame, passed back to the frame, serialized to a string. Max 4kB.q
  };
  trustedData: {
    messageBytes: string;
  };
};
```

Different client protocols may choose to include additional fields in the `untrustedData` and `trustedData` portions of the POST payload. Frame Servers may use the `clientProtocol` as a hint for what additional fields are available. All client protocols must include at least the common fields defined above.

The `of:context:$protocol_identifier` tag specifies whether it is required for a POST payload to include additional fields for the specified `clientProtocol`. If the context value is `true` or not provided, then any request to the Frame Server must include the `untrustedData` data fields for the `clientProtocol`. If the context value for a `clientProtocol` is `false` or `of:context:none` is present, then Frame server can accept requests with no additional fields required.

While the payload is similar to the [Farcaster Frames Spec](https://docs.farcaster.xyz/reference/frames/spec), it differs in two important ways. The first is the addition of the `clientProtocol` field. The second is that in place of a `timestamp`, which in Farcaster is the number of seconds since the Farcaster epoch, Open Frames uses the `unixTimestamp` field which is the number of seconds since the unix epoch.