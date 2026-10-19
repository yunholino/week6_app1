import 'dart:async';
import 'dart:isolate';
import 'package:flutter/material.dart';

void main() {
  runApp(const NoticeAssignmentApp());
}

class NoticeAssignmentApp extends StatelessWidget {
  const NoticeAssignmentApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: '공지사항 비동기 과제',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const NoticeMainScreen(),
    );
  }
}

// -----------------------------------------------------------------------------
// [도메인 모델]
// -----------------------------------------------------------------------------
class Notice {
  final String id;
  final String title;
  final String category;

  Notice({required this.id, required this.title, required this.category});
}

// -----------------------------------------------------------------------------
// [모의 서비스 (Mock Service) - 외부 패키지 없음]
// -----------------------------------------------------------------------------
class MockNoticeService {
  int fetchRequestCount = 0; // 중복 클릭 검증용 카운터
  bool shouldFailNextRequest = false;

  // AC1 & AC2: Future, async/await 기반 단발성 공지 조회
  Future<List<Notice>> fetchNotices() async {
    fetchRequestCount++;
    await Future.delayed(const Duration(seconds: 2)); // 2초 네트워크 지연 시뮬레이션

    if (shouldFailNextRequest) {
      shouldFailNextRequest = false; // 재시도 시 성공 가능하도록 플래그 리셋
      throw Exception('네트워크 연결이 불안정합니다. (모의 에러)');
    }

    return [
      Notice(id: '1', title: '[필독] 2026학년도 2학기 수강신청 안내', category: '학사'),
      Notice(id: '2', title: '플루터 비동기 프로그래밍 캡스톤 과제 제출', category: '수업'),
      Notice(id: '3', title: '중앙도서관 24시간 열람실 운영 재개', category: '시설'),
    ];
  }

  // AC3: Stream 기반 실시간 긴급 속보 스트리밍
  Stream<String> getEmergencyAlertStream() async* {
    int seq = 1;
    while (true) {
      await Future.delayed(const Duration(seconds: 3));
      yield '🚨 [실시간 긴급 속보 #$seq] 서버 정기 점검 예정 (${DateTime.now().second}초)';
      seq++;
    }
  }

  // AC4: Isolate 기반 무거운 암호화 검증 연산
  static void _isolateHeavyTask(SendPort sendPort) {
    int total = 0;
    for (int i = 0; i < 50000000; i++) {
      total += i;
    }
    sendPort.send('해시 검증 완료 ($total)');
  }

  Future<String> verifyNoticeIntegrityWithIsolate() async {
    final receivePort = ReceivePort();
    await Isolate.spawn(_isolateHeavyTask, receivePort.sendPort);
    final result = await receivePort.first as String;
    return result;
  }
}

// -----------------------------------------------------------------------------
// [메인 화면 UI]
// -----------------------------------------------------------------------------
class NoticeMainScreen extends StatefulWidget {
  const NoticeMainScreen({super.key});

  @override
  State<NoticeMainScreen> createState() => _NoticeMainScreenState();
}

class _NoticeMainScreenState extends State<NoticeMainScreen> {
  final MockNoticeService _service = MockNoticeService();

  // 상태 관리 변수
  bool _isLoading = false;
  String? _errorMessage;
  List<Notice> _notices = [];

  // Isolate 연산 상태
  bool _isIsolateProcessing = false;
  String _isolateStatus = '검증 대기 중';

  @override
  void initState() {
    super.initState();
    _loadNotices();
  }

  // AC1, AC2, AC5: 중복 요청 방지 및 로딩/성공/실패/재시도 상태 변경
  Future<void> _loadNotices() async {
    if (_isLoading) return; // 중복 클릭 요청 가드 (요청 횟수 증가 방지)

    setState(() {
      _isLoading = true;
      _errorMessage = null;
    });

    try {
      final results = await _service.fetchNotices();
      setState(() {
        _notices = results;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _errorMessage = e.toString().replaceAll('Exception: ', '');
        _isLoading = false;
      });
    }
  }

  // AC4: Isolate 연산 실행
  Future<void> _runIsolateVerification() async {
    if (_isIsolateProcessing) return;

    setState(() {
      _isIsolateProcessing = true;
      _isolateStatus = 'Isolate 백그라운드 스레드에서 무거운 연산 실행 중...';
    });

    final result = await _service.verifyNoticeIntegrityWithIsolate();

    setState(() {
      _isIsolateProcessing = false;
      _isolateStatus = result;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('공지사항 비동기 관리앱'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          // 모의 에러 발생 토글 스위치 (실패 상태 테스트용)
          IconButton(
            icon: const Icon(Icons.bug_report),
            tooltip: '다음 요청 강제 실패 설정',
            onPressed: () {
              _service.shouldFailNextRequest = true;
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(content: Text('다음 불러오기가 실패하도록 설정되었습니다.')),
              );
            },
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // -----------------------------------------------------------------
            // AC3: Stream 영역 (실시간 긴급 속보)
            // -----------------------------------------------------------------
            const Text('📡 실시간 긴급 속보 (Stream<T>)',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16)),
            const SizedBox(height: 8),
            Container(
              width: double.infinity,
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.red.shade50,
                borderRadius: BorderRadius.circular(8),
                border: Border.all(color: Colors.red.shade200),
              ),
              child: StreamBuilder<String>(
                stream: _service.getEmergencyAlertStream(),
                builder: (context, snapshot) {
                  if (snapshot.connectionState == ConnectionState.waiting) {
                    return const Text('속보 스트림 연결 중...');
                  }
                  if (snapshot.hasData) {
                    return Text(
                      snapshot.data!,
                      style: const TextStyle(
                          color: Colors.red, fontWeight: FontWeight.bold),
                    );
                  }
                  return const Text('속보 없음');
                },
              ),
            ),
            const SizedBox(height: 16),

            // -----------------------------------------------------------------
            // 분석 정보 표시 영역 (요청 횟수 및 상태 검증)
            // -----------------------------------------------------------------
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text('실제 서비스 요청 횟수: ${_service.fetchRequestCount}회',
                    style: const TextStyle(fontWeight: FontWeight.bold)),
                ElevatedButton.icon(
                  onPressed: _isLoading ? null : _loadNotices,
                  icon: const Icon(Icons.refresh),
                  label: const Text('새로고침'),
                ),
              ],
            ),
            const Divider(height: 24),

            // -----------------------------------------------------------------
            // AC1, AC2, AC5: Future / async/await 화면 상태 처리 (로딩/성공/실패)
            // -----------------------------------------------------------------
            const Text('📌 일반 공지목록 (Future / async/await)',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16)),
            const SizedBox(height: 8),
            Expanded(
              child: _buildNoticeContentArea(),
            ),
            const SizedBox(height: 16),

            // -----------------------------------------------------------------
            // AC4: Isolate 영역 (백그라운드 무거운 연산)
            // -----------------------------------------------------------------
            const Text('⚙️ 데이터 무결성 검증 (Isolate)',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16)),
            const SizedBox(height: 8),
            Container(
              width: double.infinity,
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.grey.shade100,
                borderRadius: BorderRadius.circular(8),
                border: Border.all(color: Colors.grey.shade300),
              ),
              child: Column(
                children: [
                  Text(_isolateStatus,
                      style: const TextStyle(fontSize: 13),
                      textAlign: TextAlign.center),
                  const SizedBox(height: 8),
                  ElevatedButton(
                    onPressed:
                        _isIsolateProcessing ? null : _runIsolateVerification,
                    child: _isIsolateProcessing
                        ? const SizedBox(
                            width: 16,
                            height: 16,
                            child: CircularProgressIndicator(strokeWidth: 2))
                        : const Text('Isolate 검증 실행'),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  // 화면 상태 분기 빌더
  Widget _buildNoticeContentArea() {
    // 1. 로딩 화면
    if (_isLoading) {
      return const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 12),
            Text('공지사항을 가져오는 중입니다...'),
          ],
        ),
      );
    }

    // 2. 실패 화면 (에러 발생 시 재시도 UI 제공)
    if (_errorMessage != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, color: Colors.red, size: 48),
            const SizedBox(height: 8),
            Text(_errorMessage!, style: const TextStyle(color: Colors.red)),
            const SizedBox(height: 12),
            ElevatedButton(
              onPressed: _loadNotices,
              child: const Text('재시도'),
            ),
          ],
        ),
      );
    }

    // 3. 성공 화면
    return ListView.builder(
      itemCount: _notices.length,
      itemBuilder: (context, index) {
        final notice = _notices[index];
        return Card(
          child: ListTile(
            leading: CircleAvatar(child: Text(notice.category)),
            title: Text(notice.title),
            subtitle: Text('ID: ${notice.id}'),
          ),
        );
      },
    );
  }
}
