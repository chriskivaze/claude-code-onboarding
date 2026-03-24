# Flutter Performance & UX Templates

Accessibility, performance optimization, and UX polish patterns for modern Flutter (2025/2026).

## Accessibility (A11y) Templates

### Semantic Labels on Interactive Elements

```dart
// Always wrap tappable elements with Semantics
Semantics(
  label: 'Delete workout',
  hint: 'Double tap to delete this workout entry',
  button: true,
  child: IconButton(
    icon: const Icon(Icons.delete),
    onPressed: () => _deleteWorkout(workout.id),
  ),
)

// Use semanticLabel on images
Image.network(
  user.photoUrl,
  semanticLabel: '${user.displayName} profile photo',
)

// Exclude decorative elements from semantics
Semantics(
  excludeSemantics: true,
  child: Icon(Icons.decorative_star, color: Colors.amber),
)
```

### Minimum Touch Targets (48x48 dp)

```dart
// Ensure all tappable elements meet minimum size
SizedBox(
  width: 48,
  height: 48,
  child: IconButton(
    icon: const Icon(Icons.close),
    onPressed: onClose,
  ),
)

// For custom tappable widgets
GestureDetector(
  onTap: onTap,
  child: ConstrainedBox(
    constraints: const BoxConstraints(minWidth: 48, minHeight: 48),
    child: content,
  ),
)
```

### Screen Reader Focus Order

```dart
// Use FocusTraversalGroup for logical ordering
FocusTraversalGroup(
  policy: OrderedTraversalPolicy(),
  child: Column(
    children: [
      FocusTraversalOrder(
        order: const NumericFocusOrder(1),
        child: TextField(decoration: const InputDecoration(labelText: 'Email')),
      ),
      FocusTraversalOrder(
        order: const NumericFocusOrder(2),
        child: TextField(decoration: const InputDecoration(labelText: 'Password')),
      ),
      FocusTraversalOrder(
        order: const NumericFocusOrder(3),
        child: ElevatedButton(onPressed: _submit, child: const Text('Login')),
      ),
    ],
  ),
)

// Announce dynamic changes
SemanticsService.announce('Workout saved successfully', TextDirection.ltr);
```

### Dynamic Text / Font Scaling Support

```dart
// Use MediaQuery.textScalerOf for responsive text
final textScaler = MediaQuery.textScalerOf(context);

// Never constrain text containers to fixed heights
// DO:
Text('Workout Title', style: Theme.of(context).textTheme.headlineMedium)
// DON'T:
SizedBox(height: 24, child: Text('Workout Title')) // Clips at large font sizes

// Test with: flutter run --dart-define=FLUTTER_TEXT_SCALE_FACTOR=2.0
```

## Performance Templates

### Paginated Lists

```dart
@riverpod
class PaginatedWorkouts extends _$PaginatedWorkouts {
  static const _pageSize = 20;
  DocumentSnapshot? _lastDoc;
  bool _hasMore = true;

  @override
  FutureOr<List<Workout>> build() async {
    _lastDoc = null;
    _hasMore = true;
    return _fetchPage();
  }

  Future<List<Workout>> _fetchPage() async {
    var query = FirebaseFirestore.instance
        .collection('workouts')
        .orderBy('createdAt', descending: true)
        .limit(_pageSize);

    if (_lastDoc != null) {
      query = query.startAfterDocument(_lastDoc!);
    }

    final snapshot = await query.get();
    if (snapshot.docs.length < _pageSize) _hasMore = false;
    if (snapshot.docs.isNotEmpty) _lastDoc = snapshot.docs.last;

    return snapshot.docs.map((d) => Workout.fromFirestore(d)).toList();
  }

  Future<void> loadMore() async {
    if (!_hasMore) return;
    final current = state.valueOrNull ?? [];
    final nextPage = await _fetchPage();
    state = AsyncValue.data([...current, ...nextPage]);
  }
}
```

### Image Optimization

```dart
// Always specify cacheWidth/cacheHeight for memory efficiency
CachedNetworkImage(
  imageUrl: workout.imageUrl,
  cacheKey: workout.id,
  memCacheWidth: 300,  // Match display size
  memCacheHeight: 300,
  placeholder: (_, __) => const ShimmerPlaceholder(),
  errorWidget: (_, __, ___) => const Icon(Icons.broken_image),
  fadeInDuration: const Duration(milliseconds: 200),
)
```

### Avoid Unnecessary Rebuilds

```dart
// Use const constructors everywhere possible
class WorkoutCard extends StatelessWidget {
  const WorkoutCard({super.key, required this.workout});
  final Workout workout;

  @override
  Widget build(BuildContext context) { /* ... */ }
}

// Use select() to watch specific fields only
final userName = ref.watch(
  userProvider.select((user) => user.valueOrNull?.displayName ?? ''),
);

// Wrap expensive widgets with RepaintBoundary
RepaintBoundary(
  child: CustomPaint(painter: ChartPainter(data: chartData)),
)
```

## User Experience (UX) Templates

### Haptic Feedback

```dart
import 'package:flutter/services.dart';

// On key interactions
GestureDetector(
  onTap: () {
    HapticFeedback.lightImpact();
    _onItemSelected(item);
  },
  child: itemWidget,
)

// On destructive actions
onPressed: () {
  HapticFeedback.heavyImpact();
  _showDeleteConfirmation();
}

// On success
void _onSaveSuccess() {
  HapticFeedback.mediumImpact();
  ScaffoldMessenger.of(context).showSnackBar(/* ... */);
}
```

### Skeleton / Shimmer Loading States

```dart
// Never show empty screens — always show skeleton loaders
class WorkoutListSkeleton extends StatelessWidget {
  const WorkoutListSkeleton({super.key});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 5,
      physics: const NeverScrollableScrollPhysics(),
      itemBuilder: (_, __) => Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        child: ShimmerEffect(
          child: Container(
            height: 80,
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.surfaceContainerHighest,
              borderRadius: BorderRadius.circular(16),
            ),
          ),
        ),
      ),
    );
  }
}

// Usage in AsyncValue.when()
usersAsync.when(
  loading: () => const WorkoutListSkeleton(),
  error: (err, stack) => ErrorRetryWidget(error: err, onRetry: ref.invalidate(workoutListProvider)),
  data: (workouts) => WorkoutListView(workouts: workouts),
)
```

### Shimmer Effect Widget

```dart
class ShimmerEffect extends StatefulWidget {
  final Widget child;
  const ShimmerEffect({super.key, required this.child});

  @override
  State<ShimmerEffect> createState() => _ShimmerEffectState();
}

class _ShimmerEffectState extends State<ShimmerEffect>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1500),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (_, child) => ShaderMask(
        shaderCallback: (bounds) => LinearGradient(
          begin: Alignment.centerLeft,
          end: Alignment.centerRight,
          colors: [
            Colors.grey.shade300,
            Colors.grey.shade100,
            Colors.grey.shade300,
          ],
          stops: [
            _controller.value - 0.3,
            _controller.value,
            _controller.value + 0.3,
          ],
        ).createShader(bounds),
        blendMode: BlendMode.srcATop,
        child: child,
      ),
      child: widget.child,
    );
  }
}
```

### Smooth Animations

```dart
// Page transitions with curves
PageRouteBuilder(
  pageBuilder: (_, __, ___) => const DetailScreen(),
  transitionsBuilder: (_, animation, __, child) {
    return FadeTransition(
      opacity: CurvedAnimation(
        parent: animation,
        curve: Curves.easeOutCubic,
      ),
      child: SlideTransition(
        position: Tween<Offset>(
          begin: const Offset(0, 0.05),
          end: Offset.zero,
        ).animate(CurvedAnimation(
          parent: animation,
          curve: Curves.easeOutCubic,
        )),
        child: child,
      ),
    );
  },
  transitionDuration: const Duration(milliseconds: 300),
)

// AnimatedSwitcher for state changes
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  switchInCurve: Curves.easeOutCubic,
  switchOutCurve: Curves.easeInCubic,
  child: isCompleted
      ? const Icon(Icons.check_circle, key: ValueKey('done'), color: Colors.green)
      : const Icon(Icons.circle_outlined, key: ValueKey('pending')),
)
```

### Real-time Form Validation

```dart
class WorkoutForm extends ConsumerStatefulWidget {
  const WorkoutForm({super.key});

  @override
  ConsumerState<WorkoutForm> createState() => _WorkoutFormState();
}

class _WorkoutFormState extends ConsumerState<WorkoutForm> {
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();
  bool _submitted = false;

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      autovalidateMode: _submitted
          ? AutovalidateMode.onUserInteraction
          : AutovalidateMode.disabled,
      child: Column(
        children: [
          TextFormField(
            controller: _nameController,
            decoration: const InputDecoration(
              labelText: 'Workout Name',
              hintText: 'e.g., Morning Run',
            ),
            validator: (value) {
              if (value == null || value.trim().isEmpty) {
                return 'Workout name is required';
              }
              if (value.trim().length < 3) {
                return 'Name must be at least 3 characters';
              }
              return null;
            },
          ),
          const SizedBox(height: 16),
          FilledButton(
            onPressed: _submit,
            child: const Text('Save Workout'),
          ),
        ],
      ),
    );
  }

  void _submit() {
    setState(() => _submitted = true);
    if (_formKey.currentState!.validate()) {
      HapticFeedback.mediumImpact();
      // Save workout...
    }
  }

  @override
  void dispose() {
    _nameController.dispose();
    super.dispose();
  }
}
```
