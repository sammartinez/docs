The fastest way to get started is using the `amplify-app` npx script.

<iframe src="https://www.youtube-nocookie.com/embed/wH-UnQy1ltM" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
<br/>

<amplify-block-switcher>
<amplify-block name="React">

Start with [Create React app](https://create-react-app.dev):

```bash
npx create-react-app amplify-datastore --use-npm
cd amplify-datastore
npx amplify-app@latest
```  

</amplify-block>
<amplify-block name="React Native">

Start with the [React Native CLI](https://reactnative.dev/docs/getting-started):

```bash
npx react-native init AmplifyDatastoreRN
cd AmplifyDatastoreRN
npx amplify-app@latest
npm install @react-native-community/netinfo
```

You will also need to install the pod dependencies for iOS:

```sh
npx pod-install
```
</amplify-block>
</amplify-block-switcher>
