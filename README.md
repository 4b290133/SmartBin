using System;
using System.IO.Ports;
using System.Collections.Generic;
using System.Threading.Tasks;
using System.Linq;

namespace SmartBin
{
    class Program
    {
        static SerialPort _radarPort;
        static List<int> _distHistory = new List<int>();
        static bool _isProcessing = false;

        static async Task Main(string[] args)
        {
            Console.Title = "智慧垃圾桶雷達偵測系統";
            ShowUI();

            string[] ports = SerialPort.GetPortNames();
            if (ports.Length == 0)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("[錯誤] 找不到任何序列埠裝置！請檢查雷達是否已插上 USB。");
                Console.ResetColor();
                return;
            }

            Console.WriteLine("偵測到以下裝置，請輸入編號並按 Enter：");
            for (int i = 0; i < ports.Length; i++)
            {
                Console.WriteLine($" [{i}] {ports[i]}");
            }

            int choice;
            while (!int.TryParse(Console.ReadLine(), out choice) || choice < 0 || choice >= ports.Length)
            {
                Console.Write("無效的輸入，請重新輸入編號: ");
            }

            string selectedPort = ports[choice];

            _radarPort = new SerialPort(selectedPort, 115200);
            _radarPort.DtrEnable = true;
            _radarPort.RtsEnable = true;

            try
            {
                _radarPort.Open();
                Console.Clear();
                ShowUI();
                Console.WriteLine($"已成功連線至: {selectedPort}");
                Console.WriteLine("系統運行中... (按下 Ctrl+C 可停止)");

                while (true)
                {
                    if (_radarPort.BytesToRead >= 12)
                    {
                        byte[] buffer = new byte[12];
                        _radarPort.Read(buffer, 0, 12);

                        for (int i = 0; i < 11; i++)
                        {
                            if (buffer[i] == 0x24 && buffer[i + 1] == 0x4B)
                            {
                                int currentDist = buffer[i + 2];
                                await AnalyzeGesture(currentDist);
                                break;
                            }
                        }
                    }
                    await Task.Delay(10); // 稍微暫停避免 CPU 飆高
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"\n[發生錯誤] {ex.Message}");
            }
            finally
            {
                if (_radarPort != null && _radarPort.IsOpen) _radarPort.Close();
            }
        }

        static async Task AnalyzeGesture(int dist)
        {
            _distHistory.Add(dist);
            if (_distHistory.Count > 30) _distHistory.RemoveAt(0);

            Console.SetCursorPosition(0, 8);
            string bar = new string('█', Math.Min(dist / 2, 40));
            Console.WriteLine($"[雷達偵測距離] {dist.ToString("D3")} cm |{bar.PadRight(40)}|");

            if (_distHistory.Count < 20 || _isProcessing) return;

            if (_distHistory[_distHistory.Count - 6] - dist > 25 && dist < 30)
            {
                await TriggerAction("偵測到：快速揮手", "【狀態】垃圾桶蓋已開啟", ConsoleColor.Green);
            }

            int flips = 0;
            for (int j = 2; j < _distHistory.Count; j++)
            {
                if ((_distHistory[j] > _distHistory[j - 1] && _distHistory[j - 1] < _distHistory[j - 2]) ||
                    (_distHistory[j] < _distHistory[j - 1] && _distHistory[j - 1] > _distHistory[j - 2]))
                    flips++;
            }

            if (flips >= 6 && dist > 15 && dist < 50)
            {
                await TriggerAction("偵測到：手部畫圈", "【狀態】廚餘乾燥模式啟動", ConsoleColor.Cyan);
            }
        }

        static async Task TriggerAction(string gesture, string status, ConsoleColor color)
        {
            _isProcessing = true;
            Console.SetCursorPosition(0, 12);
            Console.ForegroundColor = color;
            Console.WriteLine("==================================================");
            Console.WriteLine($"  {gesture}  ");
            Console.WriteLine($"  {status}  ");
            Console.WriteLine("==================================================");
            Console.ResetColor();

            await Task.Delay(2500);

            Console.SetCursorPosition(0, 12);
            for (int i = 0; i < 4; i++) Console.WriteLine(new string(' ', 60));

            _distHistory.Clear();
            _isProcessing = false;
        }

        static void ShowUI()
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine("==================================================");
            Console.WriteLine("        SmartBin AI - 毫米波雷達控制系統         ");
            Console.WriteLine("==================================================");
            Console.ResetColor();
        }
    }
}
