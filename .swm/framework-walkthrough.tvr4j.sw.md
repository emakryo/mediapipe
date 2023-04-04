---
id: tvr4j
title: Framework walkthrough
file_version: 1.1.2
app_version: 1.5.5
---

# 概要

`📄 mediapipe/framework`以下にあるmediapipeのパイプライン処理の実装に関して説明します。

*   CalculatorGraphのインターフェース

*   CalculatorGraphのデータ構造

*   Calculator (CalculatorNode) のライフタイム

*   Packetの入力・実行・出力まで

    <br/>

# CalculatorGraphのインターフェース

`CalculatorGraph`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:96:2:2:`class CalculatorGraph {`"/>がアプリケーションから利用されるエントリーポイントになります。主に使われるメソッドをいくつか挙げます。

## コンストラクタ

<br/>

`CalculatorGraphConfig`<swm-token data-swm-token=":mediapipe/framework/calculator.proto:224:2:2:`message CalculatorGraphConfig {`"/> で定義された設定から初期化
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
120      // Initializes the graph from its proto description (using Initialize())
121      // and crashes if something goes wrong.
122      explicit CalculatorGraph(CalculatorGraphConfig config);
123      virtual ~CalculatorGraph();
124    
```

<br/>

## 実行開始

<br/>

非同期に実行を開始する。（同期的に実行終了まで待つメソッド `Run`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:186:5:5:`  absl::Status Run() { return Run({}); }`"/> も存在する）
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
188      // Start a run of the graph.  StartRun, WaitUntilDone, Cancel, HasError,
189      // AddPacketToInputStream, and CloseInputStream allow more control over
190      // the execution of the graph run.  You can insert packets directly into
191      // a stream while the graph is running. Once StartRun has been called,
192      // the graph will continue to run until all work is either done or canceled,
193      // meaning that either WaitUntilDone() or Cancel() has been called and has
194      // completed. If StartRun returns an error, then the graph is not started and
195      // a subsequent call to StartRun can be attempted.
196      //
197      // Example:
198      //   MP_RETURN_IF_ERROR(graph.StartRun(...));
199      //   while (true) {
200      //     if (graph.HasError() || want_to_stop) break;
201      //     MP_RETURN_IF_ERROR(graph.AddPacketToInputStream(...));
202      //   }
203      //   for (const std::string& stream : streams) {
204      //     MP_RETURN_IF_ERROR(graph.CloseInputStream(stream));
205      //   }
206      //   MP_RETURN_IF_ERROR(graph.WaitUntilDone());
207      absl::Status StartRun(
208          const std::map<std::string, Packet>& extra_side_packets) {
209        return StartRun(extra_side_packets, {});
210      }
211    
```

<br/>

## 入力データの挿入

<br/>

`stream_name`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:254:14:14:`  absl::Status AddPacketToInputStream(const std::string&amp; stream_name,`"/> で指定した入力ストリームにパケットを追加
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
245      // Add a Packet to a graph input stream based on the graph input stream add
246      // mode. If the mode is ADD_IF_NOT_FULL, the packet will not be added if any
247      // queue exceeds max_queue_size specified by the graph config and will return
248      // StatusUnavailable. The WAIT_TILL_NOT_FULL mode (default) will block until
249      // the queues fall below the max_queue_size before adding the packet. If the
250      // mode is max_queue_size is -1, then the packet is added regardless of the
251      // sizes of the queues in the graph. The input stream must have been specified
252      // in the configuration as a graph level input_stream. On error, nothing is
253      // added.
254      absl::Status AddPacketToInputStream(const std::string& stream_name,
255                                          const Packet& packet);
256    
```

<br/>

## 出力データに対するコールバック

<br/>

`stream_name`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:159:8:8:`      const std::string&amp; stream_name,`"/>で指定した出力ストリームから出てきたパケットを引数に実行するコールバックの登録
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
152      // Observes the named output stream. packet_callback will be invoked on every
153      // packet emitted by the output stream. Can only be called before Run() or
154      // StartRun(). It is possible for packet_callback to be called until the
155      // object is destroyed, even if e.g. Cancel() or WaitUntilDone() have already
156      // been called. After this object is destroyed so is packet_callback.
157      // TODO: Rename to AddOutputStreamCallback.
158      absl::Status ObserveOutputStream(
159          const std::string& stream_name,
160          std::function<absl::Status(const Packet&)> packet_callback,
161          bool observe_timestamp_bounds = false);
162    
```

<br/>

# CalculatorGraphのデータ構造

CalculatorGraphの中で用いられる主なクラスに関して説明します。

[https://miro.com/app/board/uXjVMZlsp8U=/?moveToWidget=3458764549819273119&cot=14](https://miro.com/app/board/uXjVMZlsp8U=/?moveToWidget=3458764549819273119&cot=14)

## `ValidatedGraphConfig`<swm-token data-swm-token=":mediapipe/framework/validated_graph_config.h:192:2:2:`class ValidatedGraphConfig {`"/>

`CalculatorGraphConfig`<swm-token data-swm-token=":mediapipe/framework/calculator.proto:224:2:2:`message CalculatorGraphConfig {`"/>が正しいかどうか検査する。その中で GetContract を呼び出すことでCalulatorの入出力が正当かどうかのチェックも行う。

## `CalculatorNode`<swm-token data-swm-token=":mediapipe/framework/calculator_node.h:63:2:2:`class CalculatorNode {`"/>

グラフ内のノードに対応する。主なフィールドに次のようなものがある。

*   `CalculatorBase`<swm-token data-swm-token=":mediapipe/framework/calculator_base.h:75:2:2:`class CalculatorBase {`"/> Calculatorの実装

*   `InputStreamHandler`<swm-token data-swm-token=":mediapipe/framework/input_stream_handler.h:68:2:2:`class InputStreamHandler {`"/> ノードの入力ストリームの状態から実行すべきかどうかを判断する

*   `OutputStreamHandler`<swm-token data-swm-token=":mediapipe/framework/output_stream_handler.h:43:2:2:`class OutputStreamHandler {`"/> ノードの出力ストリームを次のノードの入力ストリームに伝播させる

*   `CalculatorContextManager`<swm-token data-swm-token=":mediapipe/framework/calculator_context_manager.h:36:2:2:`class CalculatorContextManager {`"/> Calculatorの実行時に参照される `CalculatorContext`<swm-token data-swm-token=":mediapipe/framework/calculator_context.h:44:2:2:`class CalculatorContext {`"/> を管理する

*   \*(ポインタ) `SchedulerQueue`<swm-token data-swm-token=":mediapipe/framework/scheduler_queue.h:38:2:2:`class SchedulerQueue : public TaskQueue {`"/> このノードが実行されるキューへの参照

## `Scheduler`<swm-token data-swm-token=":mediapipe/framework/scheduler.h:43:2:2:`class Scheduler {`"/>

ノードとノードが実行されるキューを管理する。`SchedulerQueue`<swm-token data-swm-token=":mediapipe/framework/scheduler_queue.h:38:2:2:`class SchedulerQueue : public TaskQueue {`"/> の配列（std::vector）をフィールドに持つ。

## `InputStreamManager`<swm-token data-swm-token=":mediapipe/framework/input_stream_manager.h:46:2:2:`class InputStreamManager {`"/>

ノードの入力ストリームに対応する。入力ストリームに流れるデータパケットをキューに保持し、InputStreamHandler から参照される。

## `OutputStreamManager`<swm-token data-swm-token=":mediapipe/framework/output_stream_manager.h:38:2:2:`class OutputStreamManager {`"/>

ノードの出力ストリームに対応する。次のノードのInputStreamHandlerへの参照とそのノードの中のどの入力ストリームと繋がっているかをMirrorという形で保持する。

<br/>

<br/>

This file was generated by Swimm. [Click here to view it in the app](/repos/Z2l0aHViJTNBJTNBbWVkaWFwaXBlJTNBJTNBZW1ha3J5bw==/docs/tvr4j).
