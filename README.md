centinela-app/
 ├── App.js
 ├── firebaseConfig.js
 ├── package.json
 ├── screens/
 │   ├── LoginScreen.js
 │   ├── SignUpScreen.js
 │   ├── MapScreen.js
 │   ├── CreateAlertScreen.js
 │   ├── SubscribeScreen.js
 ├── functions/
 │   └── index.js  (Webhook de Stripe)
 ├── assets/
 │   └── icon.png (Logo provisional)
 // firebaseConfig.js
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: "TU_API_KEY",
  authDomain: "TU_AUTH_DOMAIN",
  projectId: "TU_PROJECT_ID",
  storageBucket: "TU_STORAGE_BUCKET",
  messagingSenderId: "TU_SENDER_ID",
  appId: "TU_APP_ID",
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

export { auth, db };
// App.js
import React, { useEffect } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';

import LoginScreen from './screens/LoginScreen';
import SignUpScreen from './screens/SignUpScreen';
import MapScreen from './screens/MapScreen';
import CreateAlertScreen from './screens/CreateAlertScreen';
import SubscribeScreen from './screens/SubscribeScreen';

const Stack = createNativeStackNavigator();

export default function App() {
  useEffect(() => {
    const registerForPushNotifications = async () => {
      if (Device.isDevice) {
        const { status: existingStatus } = await Notifications.getPermissionsAsync();
        let finalStatus = existingStatus;
        if (existingStatus !== 'granted') {
          const { status } = await Notifications.requestPermissionsAsync();
          finalStatus = status;
        }
        if (finalStatus === 'granted') {
          const token = (await Notifications.getExpoPushTokenAsync()).data;
          console.log("Push Token:", token);
        }
      }
    };
    registerForPushNotifications();
  }, []);

  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login">
        <Stack.Screen name="Login" component={LoginScreen} />
        <Stack.Screen name="SignUp" component={SignUpScreen} />
        <Stack.Screen name="Map" component={MapScreen} />
        <Stack.Screen name="CreateAlert" component={CreateAlertScreen} />
        <Stack.Screen name="Subscribe" component={SubscribeScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
// screens/LoginScreen.js
import React, { useState } from 'react';
import { View, TextInput, Button, Text, Alert } from 'react-native';
import { auth } from '../firebaseConfig';
import { signInWithEmailAndPassword } from 'firebase/auth';

export default function LoginScreen({ navigation }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleLogin = () => {
    signInWithEmailAndPassword(auth, email, password)
      .then(() => navigation.navigate('Map'))
      .catch(error => Alert.alert('Error', error.message));
  };

  return (
    <View style={{ flex: 1, justifyContent: 'center', padding: 20 }}>
      <Text>Email</Text>
      <TextInput value={email} onChangeText={setEmail} style={{ borderWidth: 1, marginBottom: 10 }} />
      <Text>Password</Text>
      <TextInput value={password} onChangeText={setPassword} secureTextEntry style={{ borderWidth: 1, marginBottom: 10 }} />
      <Button title="Login" onPress={handleLogin} />
      <Button title="Sign Up" onPress={() => navigation.navigate('SignUp')} />
    </View>
  );
}
// screens/SignUpScreen.js
import React, { useState } from 'react';
import { View, TextInput, Button, Text, Alert } from 'react-native';
import { auth, db } from '../firebaseConfig';
import { createUserWithEmailAndPassword } from 'firebase/auth';
import { doc, setDoc } from 'firebase/firestore';

export default function SignUpScreen({ navigation }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSignUp = async () => {
    try {
      const userCredential = await createUserWithEmailAndPassword(auth, email, password);
      const user = userCredential.user;
      await setDoc(doc(db, "users", user.uid), {
        email: user.email,
        isSubscribed: false
      });
      navigation.navigate('Map');
    } catch (error) {
      Alert.alert('Error', error.message);
    }
  };

  return (
    <View style={{ flex: 1, justifyContent: 'center', padding: 20 }}>
      <Text>Email</Text>
      <TextInput value={email} onChangeText={setEmail} style={{ borderWidth: 1, marginBottom: 10 }} />
      <Text>Password</Text>
      <TextInput value={password} onChangeText={setPassword} secureTextEntry style={{ borderWidth: 1, marginBottom: 10 }} />
      <Button title="Sign Up" onPress={handleSignUp} />
    </View>
  );
}
// screens/MapScreen.js
import React, { useEffect, useState } from 'react';
import { View, Button } from 'react-native';
import MapView, { Marker } from 'react-native-maps';
import { db, auth } from '../firebaseConfig';
import { collection, getDocs, doc, getDoc } from 'firebase/firestore';
import { Alert } from 'react-native';

export default function MapScreen({ navigation }) {
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    const checkSubscription = async () => {
      const docRef = doc(db, "users", auth.currentUser.uid);
      const userSnap = await getDoc(docRef);
      if (userSnap.exists()) {
        const userData = userSnap.data();
        if (!userData.isSubscribed) {
          Alert.alert("Acceso restringido", "Suscríbete para desbloquear funciones premium.");
        }
      }
    };

    const fetchAlerts = async () => {
      const snapshot = await getDocs(collection(db, 'alerts'));
      const alertsData = snapshot.docs.map(doc => doc.data());
      setAlerts(alertsData);
    };

    checkSubscription();
    fetchAlerts();
  }, []);

  return (
    <View style={{ flex: 1 }}>
      <MapView style={{ flex: 1 }}
        initialRegion={{ latitude: 37.78825, longitude: -122.4324, latitudeDelta: 0.1, longitudeDelta: 0.1 }}>
        {alerts.map((alert, idx) => (
          <Marker
            key={idx}
            coordinate={{ latitude: alert.latitude, longitude: alert.longitude }}
            title={alert.title}
            description={alert.description}
          />
        ))}
      </MapView>
      <Button title="Crear Alerta" onPress={() => navigation.navigate('CreateAlert')} />
      <Button title="Suscribirse" onPress={() => navigation.navigate('Subscribe')} />
    </View>
  );
}
// screens/CreateAlertScreen.js
import React, { useState } from 'react';
import { View, TextInput, Button, Alert } from 'react-native';
import { db } from '../firebaseConfig';
import { collection, addDoc } from 'firebase/firestore';

export default function CreateAlertScreen({ navigation }) {
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');

  const handleCreate = async () => {
    try {
      await addDoc(collection(db, 'alerts'), {
        title,
        description,
        latitude: 37.78825,
        longitude: -122.4324
      });
      Alert.alert('Éxito', 'Alerta creada');
      navigation.goBack();
    } catch (error) {
      Alert.alert('Error', error.message);
    }
  };

  return (
    <View style={{ flex: 1, justifyContent: 'center', padding: 20 }}>
      <TextInput placeholder="Título" value={title} onChangeText={setTitle} style={{ borderWidth: 1, marginBottom: 10 }} />
      <TextInput placeholder="Descripción" value={description} onChangeText={setDescription} style={{ borderWidth: 1, marginBottom: 10 }} />
      <Button title="Crear" onPress={handleCreate} />
    </View>
  );
}
// screens/SubscribeScreen.js
import React from 'react';
import { View, Text, Button, Linking, Alert } from 'react-native';

export default function SubscribeScreen() {
  const checkoutUrl = "https://checkout.stripe.com/pay/TU_ENLACE_AQUI";

  const handleSubscribe = async () => {
    const supported = await Linking.canOpenURL(checkoutUrl);
    if (supported) {
      await Linking.openURL(checkoutUrl);
    } else {
      Alert.alert("Error", "No se pudo abrir el pago.");
    }
  };

  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>Suscríbete por $2.50/mes</Text>
      <Button title="Suscribirse" onPress={handleSubscribe} />
    </View>
  );
}
const functions = require("firebase-functions");
const admin = require("firebase-admin");
const Stripe = require("stripe");
admin.initializeApp();

const stripe = Stripe("TU_STRIPE_SECRET_KEY");

exports.handleStripeWebhook = functions.https.onRequest(async (req, res) => {
  const sig = req.headers['stripe-signature'];
  const endpointSecret = 'TU_ENDPOINT_SECRET';

  let event;
  try {
    event = stripe.webhooks.constructEvent(req.rawBody, sig, endpointSecret);
  } catch (err) {
    console.error('Webhook Error:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const userId = session.metadata.firebaseUID;

    if (userId) {
      await admin.firestore().collection('users').doc(userId).update({
        isSubscribed: true,
        subscriptionId: session.subscription
      });
      console.log(`Usuario ${userId} actualizado.`);
    }
  }

  res.status(200).send('Evento recibido');
});
