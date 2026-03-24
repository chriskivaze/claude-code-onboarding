# Flutter Code Templates

This file contains all code templates for Flutter development with Riverpod, Freezed, and Firebase.

## pubspec.yaml Essentials

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.4.0
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0
  go_router: ^14.0.0
  firebase_core: ^3.6.0
  cloud_firestore: ^5.5.0
  firebase_auth: ^5.3.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  riverpod_generator: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  build_runner: ^2.4.0
  mocktail: ^1.0.0
```

## Freezed Model Template

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'user_model.freezed.dart';
part 'user_model.g.dart';

@freezed
class UserModel with _$UserModel {
  const factory UserModel({
    required String id,
    required String email,
    required String displayName,
    @Default('') String photoUrl,
    required DateTime createdAt,
  }) = _UserModel;

  factory UserModel.fromJson(Map<String, dynamic> json) =>
      _$UserModelFromJson(json);

  factory UserModel.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return UserModel(
      id: doc.id,
      email: data['email'] ?? '',
      displayName: data['displayName'] ?? '',
      photoUrl: data['photoUrl'] ?? '',
      createdAt: (data['createdAt'] as Timestamp).toDate(),
    );
  }
}
```

## Riverpod Provider Template

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'user_provider.g.dart';

@riverpod
class UserList extends _$UserList {
  @override
  FutureOr<List<UserModel>> build() async {
    return ref.read(userRepositoryProvider).getUsers();
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(
      () => ref.read(userRepositoryProvider).getUsers(),
    );
  }

  Future<void> add(CreateUserDto dto) async {
    await ref.read(userRepositoryProvider).createUser(dto);
    await refresh();
  }
}
```

## Screen Template

```dart
class UserListScreen extends ConsumerWidget {
  const UserListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(userListProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Users')),
      body: usersAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (err, stack) => Center(child: Text('Error: $err')),
        data: (users) => ListView.builder(
          itemCount: users.length,
          itemBuilder: (context, index) {
            final user = users[index];
            return ListTile(
              title: Text(user.displayName),
              subtitle: Text(user.email),
              onTap: () => context.go('/users/${user.id}'),
            );
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.go('/users/new'),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

## GoRouter Configuration

```dart
final appRouter = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginScreen()),
    ShellRoute(
      builder: (_, __, child) => AppShell(child: child),
      routes: [
        GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
        GoRoute(path: '/users', builder: (_, __) => const UserListScreen()),
        GoRoute(path: '/users/:id', builder: (_, state) =>
          UserDetailScreen(userId: state.pathParameters['id']!)),
      ],
    ),
  ],
  redirect: (context, state) {
    final isLoggedIn = FirebaseAuth.instance.currentUser != null;
    if (!isLoggedIn && state.uri.path != '/login') return '/login';
    if (isLoggedIn && state.uri.path == '/login') return '/home';
    return null;
  },
);
```

## Widget Test Template

```dart
void main() {
  testWidgets('UserListScreen shows users', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          userListProvider.overrideWith((ref) => [
            UserModel(id: '1', email: 'a@b.com', displayName: 'Alice', createdAt: DateTime.now()),
          ]),
        ],
        child: const MaterialApp(home: UserListScreen()),
      ),
    );
    await tester.pumpAndSettle();
    expect(find.text('Alice'), findsOneWidget);
  });
}
```

## Firebase Integration Patterns

### Firestore Data Source

```dart
class UserRemoteDataSource {
  final FirebaseFirestore _firestore;

  UserRemoteDataSource(this._firestore);

  Stream<List<UserModel>> watchUsers() {
    return _firestore
        .collection('users')
        .orderBy('createdAt', descending: true)
        .snapshots()
        .map((snap) => snap.docs
            .map((doc) => UserModel.fromFirestore(doc))
            .toList());
  }

  Future<UserModel> getUser(String id) async {
    final doc = await _firestore.collection('users').doc(id).get();
    if (!doc.exists) throw Exception('User not found');
    return UserModel.fromFirestore(doc);
  }

  Future<void> createUser(UserModel user) async {
    await _firestore.collection('users').doc(user.id).set(user.toJson());
  }
}
```

### Firebase Auth Integration

```dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  Stream<User?> build() {
    return FirebaseAuth.instance.authStateChanges();
  }

  Future<void> signInWithEmail(String email, String password) async {
    await FirebaseAuth.instance.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<void> signOut() async {
    await FirebaseAuth.instance.signOut();
  }
}
```

## Clean Architecture Repository Pattern

```dart
// Domain layer - abstract repository
abstract class UserRepository {
  Stream<List<User>> watchUsers();
  Future<User> getUser(String id);
  Future<void> createUser(User user);
}

// Data layer - implementation
class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource _remoteDataSource;

  UserRepositoryImpl(this._remoteDataSource);

  @override
  Stream<List<User>> watchUsers() {
    return _remoteDataSource.watchUsers()
        .map((models) => models.map((m) => m.toEntity()).toList());
  }

  @override
  Future<User> getUser(String id) async {
    final model = await _remoteDataSource.getUser(id);
    return model.toEntity();
  }

  @override
  Future<void> createUser(User user) async {
    final model = UserModel.fromEntity(user);
    await _remoteDataSource.createUser(model);
  }
}
```
