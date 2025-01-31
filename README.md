class CameraHandler {
  late BuildContext context;
  late Function(String) onPictureTaken;
  late String base64Image;
  CameraController? _controller;
  Future<void>? _initializeControllerFuture;
  List<CameraDescription>? _cameras;
  final String logTag = 'CameraHandler';
  bool _isCameraInitialized = false;  

  CameraHandler(BuildContext context, Function(String) onPictureTaken, {String base64Image = ''}) {
    this.context = context;
    this.onPictureTaken = onPictureTaken;
    this.base64Image = base64Image;
  }

  // Initialize Camera
  Future<void> _initializeCamera() async {
    if (_isCameraInitialized) return;  

    try {
      debugPrint('$logTag: Fetching available cameras...');
      _cameras = await availableCameras();

      if (_cameras == null || _cameras!.isEmpty) {
        debugPrint('$logTag: No cameras available.');
        return;
      }

      final frontCamera = _cameras!.firstWhere(
            (camera) => camera.lensDirection == CameraLensDirection.front,
        orElse: () => throw Exception("No front camera found."),
      );

      debugPrint('$logTag: Initializing front camera: ${frontCamera.name}');
          _controller = CameraController(frontCamera, ResolutionPreset.high, enableAudio: false); 


      // Initialize the controller properly and wait for completion
      _initializeControllerFuture = _controller!.initialize();
      await _initializeControllerFuture;

      _isCameraInitialized = true;  // Mark camera as initialized

      debugPrint('$logTag: Camera initialized successfully.');
    } catch (e) {
      debugPrint('$logTag: Error initializing camera: $e');
      Fluttertoast.showToast(msg: "Error initializing camera.");
    }
  }

  // Open the camera and capture a picture
  Future<String?> openCameraAndTakePicture() async {
    try {
      WidgetsFlutterBinding.ensureInitialized();
      await _initializeCamera();

      if (_controller == null || !_controller!.value.isInitialized) {
        debugPrint('$logTag: Camera controller is not initialized.');
        return null;
      }

      // Ensure camera is fully initialized before capturing
      await _initializeControllerFuture;

      debugPrint('$logTag: Capturing image...');
      final image = await _controller!.takePicture();
      debugPrint('$logTag: Picture taken.');

      // Process Image
      img.Image? originalImage = img.decodeImage(await image.readAsBytes());
      if (originalImage != null) {
        img.Image compressedImage = img.copyResize(originalImage, width: 600);
        List<int> compressedBytes = img.encodeJpg(compressedImage, quality: 85);
        String base64Image = base64Encode(compressedBytes);

        debugPrint('$logTag: Image successfully captured and compressed.');
        return base64Image;
      } else {
        debugPrint('$logTag: Failed to decode image.');
      }
    } catch (e) {
      debugPrint('$logTag: Error capturing picture: $e');
      Fluttertoast.showToast(msg: "Error capturing picture.");
    } finally {

    }
    return null;
  }

  // Dispose Camera when done using it
  Future<void> disposeCamera() async {
    try {
      if (_controller != null && _isCameraInitialized) {
        debugPrint('$logTag: Disposing camera controller...');
        await _controller!.dispose();
        _controller = null;
        _isCameraInitialized = false;  // Reset flag
        debugPrint('$logTag: Camera disposed successfully.');
      }
    } catch (e) {
      debugPrint('$logTag: Error disposing camera: $e');
    }
  }
}
