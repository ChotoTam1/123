using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Telegram.Bot;
using Telegram.Bot.Exceptions;
using Telegram.Bot.Polling;
using Telegram.Bot.Types;
using Telegram.Bot.Types.Enums;
using System.Net.Http;
using System.Xml;
using IO = System.IO;
using System.Text.RegularExpressions;

namespace TelegramBot
{
    class Program
    {
        private static readonly string ТокенБота = "YOUR_TELEGRAM_BOT_TOKEN";
        private static readonly ITelegramBotClient КлиентБота = new TelegramBotClient(ТокенБота);
        private static readonly ConcurrentDictionary<long, СессияПользователя> СессииПользователей = new();
        private static readonly string ПутьКЛогам = "бот_лог.txt";
        private static readonly Dictionary<string, string> КэшНовостей = new();
        private static readonly string[] RssИсточники =
        {
            "https://lenta.ru/rss",
            "https://ria.ru/export/rss2/index.xml",
            "http://rss.cnn.com/rss/edition.rss",
            "https://rss.nytimes.com/services/xml/rss/nyt/HomePage.xml",
            "https://feeds.bbci.co.uk/news/rss.xml",
            "https://www.aljazeera.com/xml/rss/all.xml",
            "https://www.theguardian.com/world/rss",
            "https://e00-elmundo.uecdn.es/elmundo/rss/portada.xml",
            "https://www.lemonde.fr/rss/une.xml"
        };

        static async Task Main()
        {
            try
            {
                AppDomain.CurrentDomain.ProcessExit += (s, e) => СохранитьСессииНаДиск();
                ЗагрузитьСессииСДиска();

                var информацияОБоте = await КлиентБота.GetMeAsync();
                Журнал($"Бот запущен: @{информацияОБоте.Username}");

                using var токенОтмены = new CancellationTokenSource();
                Console.CancelKeyPress += (s, e) => { токенОтмены.Cancel(); e.Cancel = true; };
                ОчиститьСтарыеСессии();

                КлиентБота.StartReceiving(
                    ОбработатьОбновлениеAsync,
                    ОбработатьОшибкуAsync,
                    new ReceiverOptions { AllowedUpdates = new[] { UpdateType.Message, UpdateType.CallbackQuery } },
                    токенОтмены.Token
                );

                await Task.Delay(Timeout.Infinite, токенОтмены.Token);
            }
            catch (Exception ошибка)
            {
                Журнал($"Критическая ошибка при запуске: {ошибка.Message}");
            }
        }

        private static async Task ОбработатьОбновлениеAsync(ITelegramBotClient bot, Update update, CancellationToken token)
        {
            if (update.Message is { } message)
            {
                long chatId = message.Chat.Id;
                string text = message.Text?.Trim().ToLower() ?? "";

                if (text.StartsWith("/start"))
                {
                    await ОтправитьСообщениеAsync(chatId, "👋 Добро пожаловать в новостного бота! Используйте команды для получения информации:\n/webnews <ключевое слово> - поиск новостей", token);
                    return;
                }

                if (text.StartsWith("/webnews"))
                {
                    string keyword = text.Replace("/webnews", "").Trim();
                    await ПоискНовостейЧерезВеб(chatId, keyword, token);
                }
                else
                {
                    await ОтправитьСообщениеAsync(chatId, "🤖 Неизвестная команда. Используйте /start для просмотра доступных команд.", token);
                }
            }
        }

        private static async Task ПоискНовостейЧерезВеб(long идентификаторЧата, string ключевоеСлово, CancellationToken токенОтмены)
        {
            if (string.IsNullOrEmpty(ключевоеСлово))
            {
                await ОтправитьСообщениеAsync(идентификаторЧата, "❗ Укажите ключевое слово, например: /webnews технологии", токенОтмены);
                return;
            }

            if (КэшНовостей.TryGetValue(ключевоеСлово, out var кэшированныйРезультат))
            {
                await ОтправитьСообщениеAsync(идентификаторЧата, кэшированныйРезультат, токенОтмены);
                return;
            }

            var найденныеНовости = new List<string>();
            using var httpClient = new HttpClient { Timeout = TimeSpan.FromSeconds(10) };

            foreach (var rssUrl in RssИсточники)
            {
                try
                {
                    var response = await httpClient.GetAsync(rssUrl);
                    if (!response.IsSuccessStatusCode)
                    {
                        Журнал($"Ошибка {response.StatusCode} при загрузке RSS {rssUrl}");
                        continue;
                    }

                    var content = await response.Content.ReadAsStringAsync();
                    var заголовок = Regex.Match(content, "<title>(.*?)</title>").Groups[1].Value;
                    var изображение = Regex.Match(content, "<img.*?src=\"(.*?)\".*?>").Groups[1].Value;

                    найденныеНовости.Add($"📢 <b>{заголовок}</b>\nИсточник: {rssUrl}\n{(изображение != "" ? $"🖼 {изображение}" : "")}");
                }
                catch (Exception ex)
                {
                    Журнал($"Ошибка при загрузке RSS {rssUrl}: {ex.Message}");
                }
            }

            string ответ = найденныеНовости.Any()
                ? string.Join("\n\n", найденныеНовости.Take(5))
                : "❌ Новостей по вашему запросу не найдено.";

            КэшНовостей[ключевоеСлово] = ответ;
            await ОтправитьСообщениеAsync(идентификаторЧата, ответ, токенОтмены);
        }
    }
}
