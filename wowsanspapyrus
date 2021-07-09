#define _USE_MATH_DEFINES

#include <stdio.h>
#include <wchar.h>
#include <Windows.h>
#include <math.h>
#include <conio.h>
#include <locale.h>

HWND wnd;
TEXTMETRIC tmetrix;

void gotoxy(int x, int y)
{
	COORD pos = { x, y };
	SetConsoleCursorPosition(GetStdHandle(STD_OUTPUT_HANDLE), pos);
}

void setcol(unsigned char col)
{
	SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), col);
}

void CursorShow(bool show, int size)
{
	CONSOLE_CURSOR_INFO cursorInfo = { 0, };
	cursorInfo.dwSize = size;
	cursorInfo.bVisible = show;
	SetConsoleCursorInfo(GetStdHandle(STD_OUTPUT_HANDLE), &cursorInfo);
}

void InitSystem()
{
	setlocale(LC_ALL, "ko-KR");
	system("mode con lines=30 cols=80");

	DWORD prev_mode;
	GetConsoleMode(GetStdHandle(STD_INPUT_HANDLE), &prev_mode);
	SetConsoleMode(GetStdHandle(STD_INPUT_HANDLE), ENABLE_EXTENDED_FLAGS | (prev_mode & ~ENABLE_QUICK_EDIT_MODE));

	wnd = GetConsoleWindow();
	HDC dc = GetDC(wnd);
	GetTextMetrics(dc, &tmetrix);
	ReleaseDC(wnd, dc);

	LONG wLong;
	wLong = GetWindowLong(wnd, GWL_STYLE);
	SetWindowLong(wnd, GWL_STYLE, wLong & ~WS_MAXIMIZEBOX & ~WS_MINIMIZEBOX & ~WS_THICKFRAME);

	CursorShow(false, 1);
}

void GetXY(POINT* pt)
{
	GetCursorPos(pt);
	ScreenToClient(wnd, pt);
	pt->x /= tmetrix.tmAveCharWidth;
	pt->y /= tmetrix.tmHeight;
}

bool CheckClick()
{
	return GetForegroundWindow() == wnd && GetAsyncKeyState(VK_LBUTTON);
}

#define SCORE_WIDTH 32
#define SCORE_HEIGHT 24
#define SCORE_MAX 99

#define SAMPLERATE 44100

struct WAVHEADER
{
	int chunk_id;
	int chunk_size;
	int format;
	int sub1_id;
	int sub1_size;
	short audioFormat;
	short numChannels;
	int sampleRate;
	int byteRate;
	short blockAlign;
	short bitsPerSample;
	int sub2_id;
	int sub2_size;
};

#pragma comment(lib, "winmm.lib")

#define Octave 12

void* MakeWavHeader(int size)
{
	void* data = malloc(size * 16 / 8 + sizeof(WAVHEADER));

	WAVHEADER* wav = (WAVHEADER*)data;
	wav->chunk_id = 0x46464952; // 'RIFF'
	wav->format = 0x45564157; // 'WAVE'
	wav->sub1_id = 0x20746d66; // 'fmt '
	wav->sub1_size = 16;
	wav->audioFormat = 1;
	wav->numChannels = 1; // Mono 1 Stereo 2
	wav->sampleRate = SAMPLERATE; //22050 44100
	wav->bitsPerSample = 16;
	wav->blockAlign = wav->numChannels * wav->bitsPerSample / 8;
	wav->byteRate = wav->sampleRate * wav->blockAlign;
	wav->sub2_id = 0x61746164; // 'data'
	wav->sub2_size = wav->blockAlign * size;
	wav->chunk_size = wav->sub2_size + sizeof(WAVHEADER) - 8;
	return data;
}

inline int freq(int n)
{
	return 440 * pow(2, (n - 9.0) / 12);
}

inline double sinewave(int i, int j)
{
	return sin(i * M_PI / SAMPLERATE * 360 * j / 180);
}

DWORD WINAPI __BeepAsync(void* param)
{
	Beep((int)param, 200);
	return 0;
}

void BeepAsync(int freq)
{
	CreateThread(NULL, NULL, __BeepAsync, (void*)freq, NULL, NULL);
}

void MakeSound(void* data, char* note, int section, int bpm)
{
	double length = (60.0 * 4 / bpm) * SAMPLERATE;
	int offset = section * length;

	for (int i = 0; i < length; i++)
	{
		int val = 0;
		for (int j = 0; j < SCORE_HEIGHT; j++)
		{
			if (*(note + j * SCORE_WIDTH + (int)(i * SCORE_WIDTH / length)) == 1)
			{
				val += sinewave(i + offset, freq(SCORE_HEIGHT - j - 1)) * 7500;
			}
		}
		*((short*)((char*)data + sizeof(WAVHEADER)) + i + offset) = val;
	}
}

int score_pages = 1;
int current_score = 0;

bool click_activation = true;
int last_clicked_note = -1;

char* scores[SCORE_MAX];

int bpm = 120;
bool editor_sound = true;

bool play_mode = false;

void score_alloc(int page)
{
	scores[page] = (char*)calloc(SCORE_WIDTH * SCORE_HEIGHT, 1);
}

char* __pt(int page, int x, int y)
{
	return scores[page] + y * SCORE_WIDTH + x;
}

inline void score_init()
{
	score_alloc(0);
	for (int y = 0; y < SCORE_HEIGHT; y++)
	{
		for (int x = 0; x < SCORE_WIDTH; x++)
			printf("□");
		putchar('\n');
	}
}

void score_show(int page)
{
	int prev = current_score;

	for (int x = 0; x < SCORE_WIDTH; x++)
		for (int y = 0; y < SCORE_HEIGHT; y++)
			if (*__pt(prev, x, y) != *__pt(page, x, y))
			{
				gotoxy(x * 2, y);
				putwchar(*__pt(page, x, y) ? L'■' : L'□');
			}

	current_score = page;
}

void score_click(int x, int y, bool clicked)
{
	if (x < 0 || y < 0 || x >= SCORE_WIDTH * 2 || y >= SCORE_HEIGHT)
		return;

	if (!clicked)
	{
		click_activation = !(*__pt(current_score, x / 2, y));
		last_clicked_note = -1;
	}

	*__pt(current_score, x / 2, y) = click_activation;

	gotoxy(x / 2 * 2, y);
	putwchar(click_activation ? L'■' : L'□');

	if (y != last_clicked_note && click_activation && editor_sound)
	{
		BeepAsync(freq(SCORE_HEIGHT - y - 1));
	}

	last_clicked_note = y;
}

int WriteNumber(int x, int y, int len)
{
	int keycount = 0;
	int value = 0;

	CursorShow(true, 20);
	gotoxy(x, y);
	for (int i = 0; i < len; i++)
	{
		putchar(' ');
	}

	gotoxy(x, y);

	do
	{
		int key = _getch();
		if (key >= '0' && key <= '9')
		{
			putchar(key);
			value *= 10;
			value += key - '0';
			keycount++;
		}
		else if (key == ' ' || key == VK_RETURN)
			break;
	} while (keycount < len);

	CursorShow(false, 1);

	return value;
}

void ui_init()
{
	gotoxy(0, 27);
	printf("BPM 120   - PAGE 01/01 +   EDIT_SND ON");
	gotoxy(0, 28);
	printf("TEST   PLAY   STOP");
}

void ui_click(int x, int y, bool clicked)
{
	if (y < SCORE_HEIGHT || x >= SCORE_WIDTH * 2 || clicked)
		return;
	if (y == 27)
	{
		if (x >= 4 && x < 7)
		{
			int val = WriteNumber(4, 27, 3);
			if (val != 0)
			{
				bpm = val;
				gotoxy(4, 27);
			}

			printf("%03d", bpm);
		}
		else if (x == 10 && current_score > 0)
		{
			score_show(current_score - 1);
			gotoxy(17, 27);
			printf("%02d", current_score + 1);
		}
		else if (x == 23 && current_score < SCORE_MAX - 1)
		{
			if (score_pages == current_score + 1)
			{
				score_alloc(current_score + 1);
				score_pages++;
				gotoxy(20, 27);
				printf("%02d", score_pages);
			}
			score_show(current_score + 1);
			gotoxy(17, 27);
			printf("%02d", current_score + 1);
		}
		else if (x >= 36 && x <= 38 - editor_sound)
		{
			editor_sound = !editor_sound;
			gotoxy(36, 27);
			printf(editor_sound ? "ON " : "OFF");
		}
	}
	else if (y == 28)
	{
		if (x >= 0 && x < 4)
		{
			void* wav = MakeWavHeader(60.0 * 4 / bpm * SAMPLERATE + 50);
			MakeSound(wav, scores[current_score], 0, bpm);
			PlaySound((LPCWSTR)wav, NULL, SND_ASYNC | SND_MEMORY);
			free(wav);

		}
		else if (x >= 7 && x < 11)
		{
			void* wav = MakeWavHeader(60.0 * 4 / bpm * SAMPLERATE * score_pages + 50);
			for (int i = 0; i < score_pages; i++)
				MakeSound(wav, scores[i], i, bpm);
			PlaySound((LPCWSTR)wav, NULL, SND_ASYNC | SND_MEMORY);
			free(wav);
		}
		else if (x >= 14 && x < 17)
		{
			PlaySound(NULL, NULL, SND_ASYNC);
		}
	}

}

int main()
{
	POINT pt;

	unsigned char col = 0x07;
	bool clicked = false;

	InitSystem();

	score_init();
	ui_init();

	while (true)
	{
		if (CheckClick())
		{
			GetXY(&pt);
			score_click(pt.x, pt.y, clicked);
			ui_click(pt.x, pt.y, clicked);
			clicked = true;
		}
		else clicked = false;

	}
}
