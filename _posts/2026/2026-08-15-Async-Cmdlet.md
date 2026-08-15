---
title: 非同期処理をする PowerShell Cmdlet を作る
date: 2026-08-15 21:15:00+09:00
tags: PowerShell
---

PowerShell の Cmdlet で非同期処理に対応する方法。

- https://github.com/JustinGrote/ExcelFast/blob/main/Source/PowerShell/Cmdlets/TaskCmdlet.cs

某所で紹介されていて、感動したので肝の部分を解説したい。

[Sashimi] では、この方式を応用して、外部プロセスの stdout/stderr を非同期で読み取りつつ、
Cmdlet スレッド側でリアルタイムに出力する仕組みを構築している。

## Cmdlet は非同期出力不可
PowerShell の Cmdlet は同期的に動く。

1. Begin (BeginProcessing) で初期化
2. Process (ProcessRecord) でパイプライン入力処理
3. End (EndProcessing) で最終的な処理

のような流れで、PowerShell 本体から呼び出される。

各処理で非同期処理も可能ではあるものの、出力処理（`WriteObject` とか `WriteError` とか）はそのCmdletのメインスレッドからしか実行できない制約がある。

```csharp
[Cmdlet(VerbsDiagnostic.Test, "AsyncCmd")]
public class TestAsyncCommand : PSCmdlet
{
    [Parameter()]
    public string Message { get; set; } = "Hello";

    private Task? _outputTask;
    private Stopwatch _sw = Stopwatch.StartNew();

    protected override void BeginProcessing()
    {
        WriteVerbose($"({_sw.Elapsed}) Start: BeginProcessing");
        _outputTask = Task.Run(OutputAsync);
        WriteVerbose($"({_sw.Elapsed}) End: BeginProcessing");
    }
    protected override void EndProcessing()
    {
        WriteVerbose($"({_sw.Elapsed}) Start: EndProcessing");
        try
        {
            _outputTask?.Wait();
        }
        catch (AggregateException ex)
        {
            WriteError(new(ex.InnerException, "AsyncError", ErrorCategory.NotSpecified, this));
        }
        WriteVerbose($"({_sw.Elapsed}) End: EndProcessing");
    }

    private async Task OutputAsync()
    {
        for (var i = 0; i < 10; i++)
        {
            WriteObject($"({_sw.Elapsed}) {Message}");
            await Task.Delay(100);
        }
    }
}
```

上記のように、 `BeginProcessing` 内で非同期処理を走らせ、`EndProcessing` で終了を待とうとするとエラーになる。

```
Test-AsyncCmd: The WriteObject and WriteError methods cannot be called from outside the overrides of the BeginProcessing, ProcessRecord,
and EndProcessing methods, and they can only be called from within the same thread.
Validate that the cmdlet makes these calls correctly, or contact Microsoft Customer Support Services.
```

PowerShell の `WriteObject` は内部で「呼び出し元スレッド ID」を検証しており、Cmdlet 実行スレッド以外からの呼び出しを明確に禁止しているのだ。
出力系メソッドは、メインスレッドから呼び出さなくてはならない。

どうするか？

出力させたいものはキューに貯めて、後で出力する。

## 修正案

以下のようになる。

```csharp
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Management.Automation;

[Cmdlet(VerbsDiagnostic.Test, "AsyncCmd")]
public class TestAsyncCommand : PSCmdlet
{
    [Parameter()]
    public string Message { get; set; } = "Hello";

    private enum RecordType { Output, Error, Warning, Verbose, Debug, Information }
    private readonly record struct OutputRecord(RecordType Type, object Value);

    private BlockingCollection<OutputRecord> _outputs = new();
    private Task? _outputTask;
    private Stopwatch _sw = Stopwatch.StartNew();

    protected override void BeginProcessing()
    {
        _outputs.Add(new(RecordType.Verbose, $"({_sw.Elapsed}) Start: BeginProcessing"));
        _outputTask = Task.Run(OutputAsync);
        _outputs.Add(new(RecordType.Verbose, $"({_sw.Elapsed}) End: BeginProcessing"));
    }
    protected override void EndProcessing()
    {
        _outputs.Add(new(RecordType.Verbose, $"({_sw.Elapsed}) Start: EndProcessing"));

        Task _completedTask = Task.Run(async () =>
        {
            try
            {
                await _outputTask;
            }
            finally
            {
                _outputs.CompleteAdding();
            }
        });

        foreach (var record in _outputs.GetConsumingEnumerable())
        {
            switch (record.Type)
            {
                case RecordType.Output:
                    WriteObject(record.Value);
                    break;
                case RecordType.Error:
                    if (record.Value is ErrorRecord err)
                        WriteError(err);
                    else if (record.Value is Exception ex)
                        WriteError(new ErrorRecord(ex, "CmdletError", ErrorCategory.NotSpecified, this));
                    break;
                case RecordType.Warning:
                    WriteWarning($"{record.Value}");
                    break;
                case RecordType.Verbose:
                    WriteVerbose($"{record.Value}");
                    break;
                case RecordType.Debug:
                    WriteDebug($"{record.Value}");
                    break;
                case RecordType.Information:
                    if (record.Value is InformationRecord info)
                        WriteInformation(info);
                    else
                        WriteInformation(record.Value, []);
                    break;
            }
        }
        try
        {
            _completedTask.Wait();
        }
        catch (AggregateException ex)
        {
            WriteError(new(ex.InnerException, "AsyncError", ErrorCategory.NotSpecified, this));
        }
        finally
        {
            _outputs.Dispose();
        }
        WriteVerbose($"({_sw.Elapsed}) End: EndProcessing");
    }

    private async Task OutputAsync()
    {
        for (var i = 0; i < 10; i++)
        {
            _outputs.Add(new OutputRecord(RecordType.Output, $"({_sw.Elapsed}) {Message}"));
            await Task.Delay(100);
        }
    }
}
```

### 解説

キューにはスレッドセーフな `BlockingCollection<T>` を使用するのが肝。

```csharp
    private BlockingCollection<OutputRecord> _outputs = new();
```

非同期メソッド内では、この `_output` に追加していく。

キューに貯める「生産者」を別スレッド、溜まったオブジェクトを使用する「消費者」を Cmdletのスレッドにするのだ。

スレッドセーフなキューとして、`ConcurrentQueue<T>` があるが、ただ `TryDequeue()` していくだけだと空になった後に追加で溜まるかもしれないものを待つのに別実装が必要になる。
対して `BlockingCollection<T>` は `GetConsumingEnumerable()` メソッドで生産完了フラグが立つまで待つことができるのだ。

```csharp
    protected override void EndProcessing()
    {
        // キュー生産者側を別スレッドで
        Task _completedTask = Task.Run(async () =>
        {
            try
            {
                await _outputTask;
            }
            finally
            {
                _outputs.CompleteAdding();
            }
        });

        // キューの消費者側は Cmdlet のスレッドで
        foreach (var record in _outputs.GetConsumingEnumerable())
        {
            // WriteObject() 等
        }

        // 最後の終了処理
        try
        {
            _completedTask.Wait();
        }
        catch (AggregateException ex)
        {
            WriteError(new(ex.InnerException, "AsyncError", ErrorCategory.NotSpecified, this));
        }
        finally
        {
            _outputs.Dispose();
        }
    }
```
`_outputs.GetConsumingEnumerable()` 中は生産完了フラグが立つまで完全に待ちになるので、別スレッドから出力タスクの完了と `_outputs.CompleteAdding()` による生産完了通知をする必要がある点に注意。


また、溜め込むオブジェクトは `WriteObject`, `WriteError`, `WriteWarning` 等、どのストリームに出力するか区別できるようにしておくと、良い。
```csharp
    private enum RecordType { Output, Error, Warning, Verbose, Debug, Information }
    private readonly record struct OutputRecord(RecordType Type, object Value);

```

```csharp
        foreach (var record in _outputs.GetConsumingEnumerable())
        {
            switch (record.Type)
            {
                case RecordType.Output:
                    WriteObject(record.Value);
                    break;
                case RecordType.Error:
                    WriteError(...);
                    break;
                case RecordType.Warning:
                    WriteWarning(...);
                    break;
                case RecordType.Verbose:
                    WriteVerbose(...);
                    break;
                case RecordType.Debug:
                    WriteDebug(...);
                    break;
                case RecordType.Information:
                    WriteInformation(...);
                    break;
            }
        }
```

---

PowerShell の Cmdlet は本質的に同期的だが、
非同期処理を「Cmdlet スレッドに戻す」ことで非同期出力を実現できる。
`BlockingCollection<T>` を使ったこの方式は、PowerShell の制約を自然に回避する最も堅牢な方法のひとつだ。

[Sashimi]: https://github.com/teramako/Sashimi/ "teramako/Sashimi: Raw process I/O for PowerShell — execute external commands and handle stdin/stdout/stderr as byte streams."
