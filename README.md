import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:url_launcher/url_launcher.dart';
import 'package:fluttertoast/fluttertoast.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const SobironShopApp());
}

class SobironShopApp extends StatelessWidget {
  const SobironShopApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Sobiron Shop',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        brightness: Brightness.dark,
        primaryColor: Colors.green,
        scaffoldBackgroundColor: Colors.black,
        appBarTheme: const AppBarTheme(
          backgroundColor: Colors.black,
          foregroundColor: Colors.white,
        ),
        colorScheme: ColorScheme.dark(
          primary: Colors.greenAccent.shade400,
          secondary: Colors.greenAccent.shade200,
        ),
        useMaterial3: true,
      ),
      home: const SplashScreen(),
    );
  }
}

/// ---------------- Splash Screen ----------------
class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  @override
  void initState() {
    super.initState();
    Future.delayed(const Duration(seconds: 2), () {
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(builder: (_) => const HomePage()),
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
          const Icon(Icons.shopping_bag, size: 80, color: Colors.greenAccent),
          const SizedBox(height: 16),
          const Text('Sobiron Shop',
              style: TextStyle(
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                  color: Colors.white)),
          const SizedBox(height: 8),
          Text('Your Natural Store 🌿',
              style: TextStyle(color: Colors.greenAccent.shade200)),
        ]),
      ),
    );
  }
}

/// ---------------- Home Page ----------------
class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final firestore = FirebaseFirestore.instance;
  final auth = FirebaseAuth.instance;
  final String adminEmail = "mzihanhossen@gmail.com";
  final String whatsapp = "8801868352410";

  @override
  Widget build(BuildContext context) {
    final isAdmin = auth.currentUser?.email == adminEmail;
    return Scaffold(
      appBar: AppBar(
        title: const Text('Sobiron Shop'),
        actions: [
          IconButton(
            icon: const Icon(Icons.person),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const AuthScreen()),
            ),
          ),
          if (isAdmin)
            IconButton(
              icon: const Icon(Icons.admin_panel_settings),
              onPressed: () => Navigator.push(
                context,
                MaterialPageRoute(builder: (_) => const AdminPanel()),
              ),
            ),
        ],
      ),
      body: StreamBuilder<QuerySnapshot>(
        stream: firestore.collection('products').snapshots(),
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }
          final products = snapshot.data!.docs;
          return GridView.builder(
            padding: const EdgeInsets.all(10),
            gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 2,
              mainAxisSpacing: 12,
              crossAxisSpacing: 12,
              childAspectRatio: 0.75,
            ),
            itemCount: products.length,
            itemBuilder: (context, index) {
              final data = products[index].data() as Map<String, dynamic>;
              return ProductCard(
                id: products[index].id,
                name: data['name'] ?? '',
                price: data['price'] ?? 0,
                image: data['image'] ?? '',
                desc: data['desc'] ?? '',
                onBuy: () => orderViaWhatsApp(data['name'], data['price']),
              );
            },
          );
        },
      ),
    );
  }

  Future<void> orderViaWhatsApp(String name, int price) async {
    final msg = Uri.encodeComponent(
        "Hello Sobiron Shop! I want to order \"$name\" (৳$price).");
    final url = "https://wa.me/$whatsapp?text=$msg";
    if (await canLaunchUrl(Uri.parse(url))) {
      await launchUrl(Uri.parse(url), mode: LaunchMode.externalApplication);
    } else {
      Fluttertoast.showToast(msg: "Couldn't open WhatsApp!");
    }
  }
}

/// ---------------- Product Card ----------------
class ProductCard extends StatelessWidget {
  final String id, name, image, desc;
  final int price;
  final VoidCallback onBuy;

  const ProductCard({
    super.key,
    required this.id,
    required this.name,
    required this.price,
    required this.image,
    required this.desc,
    required this.onBuy,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      color: Colors.grey.shade900,
      child: Column(
        children: [
          Expanded(
            child: image.isNotEmpty
                ? Image.network(image, fit: BoxFit.cover)
                : Container(color: Colors.grey.shade800),
          ),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Column(children: [
              Text(name,
                  style: const TextStyle(
                      color: Colors.white, fontWeight: FontWeight.bold)),
              Text("৳$price",
                  style: const TextStyle(color: Colors.greenAccent)),
              const SizedBox(height: 6),
              ElevatedButton(
                onPressed: onBuy,
                style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.greenAccent.shade400),
                child: const Text("Order Now"),
              )
            ]),
          )
        ],
      ),
    );
  }
}

/// ---------------- Auth Screen ----------------
class AuthScreen extends StatefulWidget {
  const AuthScreen({super.key});

  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen> {
  final auth = FirebaseAuth.instance;
  final email = TextEditingController();
  final pass = TextEditingController();
  bool login = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(login ? "Login" : "Register")),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(children: [
          TextField(
              controller: email,
              decoration: const InputDecoration(labelText: "Email")),
          TextField(
              controller: pass,
              obscureText: true,
              decoration: const InputDecoration(labelText: "Password")),
          const SizedBox(height: 20),
          ElevatedButton(
              onPressed: handleAuth,
              child: Text(login ? "Login" : "Register")),
          TextButton(
              onPressed: () => setState(() => login = !login),
              child: Text(login
                  ? "Create new account"
                  : "Already have an account? Login"))
        ]),
      ),
    );
  }

  Future<void> handleAuth() async {
    try {
      if (login) {
        await auth.signInWithEmailAndPassword(
            email: email.text.trim(), password: pass.text.trim());
      } else {
        await auth.createUserWithEmailAndPassword(
            email: email.text.trim(), password: pass.text.trim());
      }
      Fluttertoast.showToast(msg: "Success!");
      if (mounted) Navigator.pop(context);
    } catch (e) {
      Fluttertoast.showToast(msg: e.toString());
    }
  }
}

/// ---------------- Admin Panel ----------------
class AdminPanel extends StatefulWidget {
  const AdminPanel({super.key});

  @override
  State<AdminPanel> createState() => _AdminPanelState();
}

class _AdminPanelState extends State<AdminPanel> {
  final firestore = FirebaseFirestore.instance;
  final name = TextEditingController();
  final price = TextEditingController();
  final image = TextEditingController();
  final desc = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Admin Panel")),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: ListView(children: [
          const Text("Add Product",
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          TextField(controller: name, decoration: const InputDecoration(labelText: "Name")),
          TextField(controller: desc, decoration: const InputDecoration(labelText: "Description")),
          TextField(controller: price, decoration: const InputDecoration(labelText: "Price"), keyboardType: TextInputType.number),
          TextField(controller: image, decoration: const InputDecoration(labelText: "Image URL")),
          const SizedBox(height: 12),
          ElevatedButton(
            onPressed: addProduct,
            style: ElevatedButton.styleFrom(backgroundColor: Colors.greenAccent.shade400),
            child: const Text("Add Product"),
          ),
          const Divider(height: 30),
          const Text("All Products",
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          StreamBuilder<QuerySnapshot>(
            stream: firestore.collection('products').snapshots(),
            builder: (context, snapshot) {
              if (!snapshot.hasData) return const CircularProgressIndicator();
              final docs = snapshot.data!.docs;
              return Column(
                children: docs.map((doc) {
                  final data = doc.data() as Map<String, dynamic>;
                  return ListTile(
                    title: Text(data['name'] ?? ''),
                    subtitle: Text("৳${data['price']}"),
                    trailing: IconButton(
                        icon: const Icon(Icons.delete, color: Colors.red),
                        onPressed: () =>
                            firestore.collection('products').doc(doc.id).delete()),
                  );
                }).toList(),
              );
            },
          ),
        ]),
      ),
    );
  }

  Future<void> addProduct() async {
    try {
      await firestore.collection('products').add({
        "name": name.text.trim(),
        "desc": desc.text.trim(),
        "price": int.tryParse(price.text.trim()) ?? 0,
        "image": image.text.trim(),
        "created": FieldValue.serverTimestamp(),
      });
      Fluttertoast.showToast(msg: "Product Added!");
      name.clear();
      desc.clear();
      price.clear();
      image.clear();
    } catch (e) {
      Fluttertoast.showToast(msg: e.toString());
    }
  }
}

