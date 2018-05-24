# Лекция 4

Темы, которые будут рассмотрены в лекции:

- Организация работы с шейдерами;
- Index Buffers;
- Отладка в OpenGL;

## К вопросу об организации работы с шейдерами

В вашем проекте может возникнуть необходимость использования нескольких различных vertex- и fragment-шейдеров. К тому же, в любом в OpenGL необходимо разделение на vertex- и fragment-шейдеры, так что нам понадобятся в самом простейшем случае как минимум два шейдера. Возникает вопрос об организации работы с исходным кодом шейдеров. Крайне неудобно хранить их в качестве встроенной константы внутри кода одного из исходных файлов проекта, как мы это делали раньше. Давайте выделим наши шейдеры в отдельный файл (например, Basic.shader), при этом обозначив, где находится vertex-shader, а где — fragment-shader:

```C++
#shader vertex
#version 330 core

layout(location = 0) in vec4 position;

void main() {
	gl_Position = position;
};

#shader fragment
#version 330 core

layout(location = 0) out vec4 color;

void main() {
	color = vec4(1.0, 0.0, 0.0, 1.0);
};
```

И удалим из исходного файла строковые константы, в которых мы записали эти шейдеры:

```C++
	std::string vertexShader =
		"#version 330 core\n"
		"\n"
		"layout(location = 0) in vec4 position;"
		"\n"
		"void main() {\n"
		"	gl_Position = position;\n"
		"}\n";

	std::string fragmentShader =
		"#version 330 core\n"
		"\n"
		"layout(location = 0) out vec4 color;"
		"\n"
		"void main() {\n"
		"	color = vec4(1.0, 0.0, 0.0, 1.0);\n"
		"}\n";
```

Теперь нам необходимо каким-то образом извлечь эти шейдеры из файла. Для этого напишем функцию:

```C++
struct ShaderProgramSource {
	std::string VertexSource;
	std::string FragmentSource;
};

static ShaderProgramSource ParseShader(const std::string& filepath) {
	std::ifstream stream(filepath);

	enum class ShaderType {
		NONE = -1, VERTEX = 0, FRAGMENT = 1
	};

	std::stringstream ss[2];
	std::string line;
	ShaderType type = ShaderType::NONE;
	while (getline(stream, line)) {
		if (line.find("#shader") != std::string::npos) {
			if (line.find("vertex") != std::string::npos) {
				// здесь мы подгружаем vertex shader
				type = ShaderType::VERTEX;
			} else if (line.find("fragment") != std::string::npos) {
				// здесь мы подгружаем fragment shader
				type = ShaderType::FRAGMENT;
			}
		} else {
			ss[(int)type] << line << "\n";
		}
	}

	return { ss[0].str(), ss[1].str() };
}
```

(Весь код предназначен для демонстрации взаимодейстивя с OpenGL, поэтому не будем особо заморачиваться различными проверками и т.д.)

Нам понадобится подключить заголовочные файлы fstream, string и sstream:

```C++
#include <fstream>
#include <string>
#include <sstream>
```

Ну и теперь изменм фрагмент кода, который отвечал за загрузку шейдеров:

```C++
ShaderProgramSource source = ParseShader("Basic.shader");
std::cout << "VERTEX" << std::endl;
std::cout << source.VertexSource << std::endl;
std::cout << "FRAGMENT" << std::endl;
std::cout << source.FragmentSource << std::endl;

unsigned int shader = CreateShader(source.VertexSource, source.FragmentSource);
glUseProgram(shader);
```

После всех преобразований, файл main.cpp будет выглядеть следующим образом:

```C++
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

struct ShaderProgramSource {
	std::string VertexSource;
	std::string FragmentSource;
};

static ShaderProgramSource ParseShader(const std::string& filepath) {
	std::ifstream stream(filepath);

	enum class ShaderType {
		NONE = -1, VERTEX = 0, FRAGMENT = 1
	};

	std::stringstream ss[2];
	std::string line;
	ShaderType type = ShaderType::NONE;
	while (getline(stream, line)) {
		if (line.find("#shader") != std::string::npos) {
			if (line.find("vertex") != std::string::npos) {
				// здесь мы подгружаем vertex shader
				type = ShaderType::VERTEX;
			} else if (line.find("fragment") != std::string::npos) {
				// здесь мы подгружаем fragment shader
				type = ShaderType::FRAGMENT;
			}
		} else {
			ss[(int)type] << line << "\n";
		}
	}

	return { ss[0].str(), ss[1].str() };
}

static unsigned int CompileShader(unsigned int type, const std::string& source) {
	unsigned int id = glCreateShader(type);
	const char* src = source.c_str();
	glShaderSource(id, 1, &src, nullptr);
	glCompileShader(id);

	int result;
	glGetShaderiv(id, GL_COMPILE_STATUS, &result);
	if (result == GL_FALSE) {
		int length;
		glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
		char* message = (char *) alloca(length * sizeof(char));
		glGetShaderInfoLog(id, length, &length, message);
		std::cout << "Failed to compile " << 
			(type == GL_VERTEX_SHADER ? "vertex" : "fragment") << " shader!" << std::endl;
		std::cout << message << std::endl;
		glDeleteShader(id);
		return 0;
	}

	return id;
}

static int CreateShader(const std::string& vertexShader, const std::string& fragmentShader) {
	unsigned int program = glCreateProgram();
	unsigned int vs = CompileShader(GL_VERTEX_SHADER, vertexShader);
	unsigned int fs = CompileShader(GL_FRAGMENT_SHADER, fragmentShader);
	glAttachShader(program, vs);
	glAttachShader(program, fs);
	glLinkProgram(program);
	glValidateProgram(program);

	glDeleteShader(vs);
	glDeleteShader(fs);

	return program;
}

int main(void)
{
	GLFWwindow* window;

	/* Initialize the library */
	if (!glfwInit())
		return -1;

	/* Create a windowed mode window and its OpenGL context */
	window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
	if (!window) {
		glfwTerminate();
		return -1;
	}

	/* Make the window's context current */
	glfwMakeContextCurrent(window);

	if (glewInit() != GLEW_OK) {
		std::cout << "Error: couldn't connect GLEW" << std::endl;
	}

	float positions[6] = {
		-0.5f,	-0.5f,
		 0.0f,	 0.5f,
		 0.5f,	-0.5f
	};

	unsigned int buffer;
	glGenBuffers(1, &buffer);
	glBindBuffer(GL_ARRAY_BUFFER, buffer);

	glEnableVertexAttribArray(0);
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), 0);

	ShaderProgramSource source = ParseShader("Basic.shader");
	std::cout << "VERTEX" << std::endl;
	std::cout << source.VertexSource << std::endl;
	std::cout << "FRAGMENT" << std::endl;
	std::cout << source.FragmentSource << std::endl;

	unsigned int shader = CreateShader(source.VertexSource, source.FragmentSource);
	glUseProgram(shader);
	
	glBufferData(GL_ARRAY_BUFFER, 6 * sizeof(float), positions, GL_STATIC_DRAW);

	/* Loop until the user closes the window */
	while (!glfwWindowShouldClose(window))
	{
		/* Render here */
		glClear(GL_COLOR_BUFFER_BIT);

		glDrawArrays(GL_TRIANGLES, 0, 3);

		/* Swap front and back buffers */
		glfwSwapBuffers(window);

		/* Poll for and process events */
		glfwPollEvents();
	}

	glDeleteProgram(shader);
	glfwTerminate();
	return 0;
}
```

И теперь у нас есть отдельный файл с шейдерами, что обеспечит более комфортную работу как с точки зрения организации кода, так и с точки зрения работы с системой контроля версий и т.д. В приципе, ничего не мешает структурировать ваши файлы с шейдерами как угодно, разделять файлы с vertex и fragment шейдерами и т.п. это вопрос собственных предпочтений.

## Index Buffers

Все это время мы занимались непонятно чем, рисовали простые треугольники и вообще все это выглядело достаточно примитивно. Но что будет если попытаться перейти к реальным задачам, связанным с компьютерной графикой и, скажем, попытаться нарисовать квадрат. Если отставить шутки в сторону, то основной примитив, с котором мы будем сталкиваться при работе с компьютерной графикой — треугольник. Любая более сложная фигура может быть представлена в виде совокупности различных треугольников. Например, если мы говорим о квадрате, то он представим в виде двух равных прямоугольных равнобедренных треугольников с совпадающими гипотенузами. Поменяем размерность нашего окна на 480x480, чтобы не думать о соотношении сторон и изменим массив с позициями, чтобы изобразить правую нижнюю половину квадрата:

```C++
window = glfwCreateWindow(480, 480, "Hello World", NULL, NULL);
if (!window) {
	glfwTerminate();
	return -1;
}

/* Make the window's context current */
glfwMakeContextCurrent(window);

if (glewInit() != GLEW_OK) {
	std::cout << "Error: couldn't connect GLEW" << std::endl;
}

float positions[6] = {
	-0.5f,	-0.5f,
	 0.5f,	 -0.5f,
	 0.5f,	0.5f
};
```

Добавим еще три вершины, которые позоволят построить вторую половину квадрата:

```C++
float positions[] = {
	-0.5f,	-0.5f,
	 0.5f,	 -0.5f,
	 0.5f,	0.5f,
	 -0.5f,	0.5f,
};
```

И увеличим размер нашего буфера, чтобы отрисовывались все 6 вершин:

```C++
glBufferData(GL_ARRAY_BUFFER, 6 * 2 * sizeof(float), positions, GL_STATIC_DRAW);
```

Неплохо!

Но как можно заметить, мы не очень эффективно используем нашу память, поскольку из 6 точек уникальных лишь 4. Индексный буфер (Index Buffers) — инструмент, позволяющий решить эту задачу. Конечно в случае с квадратом, экономия памяти не выглядит так драматично, но что если у вас отрисовывается 3d модель с миллионами полигонов? К тому же сами вершины могут иметь значительно более сложную структуру, чем просто координаты.

Так что давайте перейдем к использованию индексного буфера.

Запишем в массив с вершинами только уникальные позиции:

```C++
float positions[] = {
	-0.5f,	-0.5f,
	 0.5f,	 -0.5f,
	 0.5f,	0.5f,
	 -0.5f,	0.5f,
};
```

И добавим массив, который описывает наши треугольники:

```C++
unsigned int indices[] = {
	0, 1, 2,
	2, 3, 0,
};
```

Создадим индексный буфер:

```C++
unsigned int ibo;
glGenBuffers(1, &ibo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, 6 * sizeof(unsigned int), indices, GL_STATIC_DRAW);
```

И изменим вызов функции glDrawArrays на glDrawElements:

```C++
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, nullptr);
```

Итоговый текст исходного файла:

```C++
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

struct ShaderProgramSource {
	std::string VertexSource;
	std::string FragmentSource;
};

static ShaderProgramSource ParseShader(const std::string& filepath) {
	std::ifstream stream(filepath);

	enum class ShaderType {
		NONE = -1, VERTEX = 0, FRAGMENT = 1
	};

	std::stringstream ss[2];
	std::string line;
	ShaderType type = ShaderType::NONE;
	while (getline(stream, line)) {
		if (line.find("#shader") != std::string::npos) {
			if (line.find("vertex") != std::string::npos) {
				// здесь мы подгружаем vertex shader
				type = ShaderType::VERTEX;
			} else if (line.find("fragment") != std::string::npos) {
				// здесь мы подгружаем fragment shader
				type = ShaderType::FRAGMENT;
			}
		} else {
			ss[(int)type] << line << "\n";
		}
	}

	return { ss[0].str(), ss[1].str() };
}

static unsigned int CompileShader(unsigned int type, const std::string& source) {
	unsigned int id = glCreateShader(type);
	const char* src = source.c_str();
	glShaderSource(id, 1, &src, nullptr);
	glCompileShader(id);

	int result;
	glGetShaderiv(id, GL_COMPILE_STATUS, &result);
	if (result == GL_FALSE) {
		int length;
		glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
		char* message = (char *) alloca(length * sizeof(char));
		glGetShaderInfoLog(id, length, &length, message);
		std::cout << "Failed to compile " << 
			(type == GL_VERTEX_SHADER ? "vertex" : "fragment") << " shader!" << std::endl;
		std::cout << message << std::endl;
		glDeleteShader(id);
		return 0;
	}

	return id;
}

static int CreateShader(const std::string& vertexShader, const std::string& fragmentShader) {
	unsigned int program = glCreateProgram();
	unsigned int vs = CompileShader(GL_VERTEX_SHADER, vertexShader);
	unsigned int fs = CompileShader(GL_FRAGMENT_SHADER, fragmentShader);
	glAttachShader(program, vs);
	glAttachShader(program, fs);
	glLinkProgram(program);
	glValidateProgram(program);

	glDeleteShader(vs);
	glDeleteShader(fs);

	return program;
}

int main(void)
{
	GLFWwindow* window;

	/* Initialize the library */
	if (!glfwInit())
		return -1;

	/* Create a windowed mode window and its OpenGL context */
	window = glfwCreateWindow(480, 480, "Hello World", NULL, NULL);
	if (!window) {
		glfwTerminate();
		return -1;
	}

	/* Make the window's context current */
	glfwMakeContextCurrent(window);

	if (glewInit() != GLEW_OK) {
		std::cout << "Error: couldn't connect GLEW" << std::endl;
	}

	float positions[] = {
		-0.5f,	-0.5f,
		 0.5f,	 -0.5f,
		 0.5f,	0.5f,
		 -0.5f,	0.5f,
	};

	unsigned int indices[] = {
		0, 1, 2,
		2, 3, 0,
	};

	unsigned int buffer;
	glGenBuffers(1, &buffer);
	glBindBuffer(GL_ARRAY_BUFFER, buffer);
	glBufferData(GL_ARRAY_BUFFER, 4 * 2 * sizeof(float), positions, GL_STATIC_DRAW);

	unsigned int ibo;
	glGenBuffers(1, &ibo);
	glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);
	glBufferData(GL_ELEMENT_ARRAY_BUFFER, 6 * sizeof(unsigned int), indices, GL_STATIC_DRAW);

	glEnableVertexAttribArray(0);
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), 0);

	ShaderProgramSource source = ParseShader("Basic.shader");
	std::cout << "VERTEX" << std::endl;
	std::cout << source.VertexSource << std::endl;
	std::cout << "FRAGMENT" << std::endl;
	std::cout << source.FragmentSource << std::endl;

	unsigned int shader = CreateShader(source.VertexSource, source.FragmentSource);
	glUseProgram(shader);

	/* Loop until the user closes the window */
	while (!glfwWindowShouldClose(window))
	{
		/* Render here */
		glClear(GL_COLOR_BUFFER_BIT);

		glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, nullptr);

		/* Swap front and back buffers */
		glfwSwapBuffers(window);

		/* Poll for and process events */
		glfwPollEvents();
	}

	glDeleteProgram(shader);
	glfwTerminate();
	return 0;
}
```

## Отладка в OpenGL

Если не озабаться правильной обработкой ошибок в OpenGL, то вы часто будете сталкиваться с ситуацией, когда ваша программа не работает, ничего не отображается и в консоль ничего не записывается. Как тогда отлаживать программу? Как облегчить процесс отладки?

Рассмотрим инструменты, которые OpenGL предоставляет для обработки ошибок. Первый способ это вызов функции `glGetError`, которая возвращает код последней ошибки. Именно тот факт, что возвращается код только последней ошибки и делает данный способ достаточно неудобным, поскольку если при одном вызове произошло несколько ошибок, вы это не сможете продиагностировать. Рассмотрим работу glGetError на примере. Изменим тип значения в массиве в вызове функции glDrawElements:

```C++
glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr);
```

После этого наша программа перестанет корректно выполнятся: квадрат не будет отрисовываться и в тоже время в консоль не будет выведена никакая диагностическая информация. Как же понять, что именно произошло?

Чтобы понять, приводит ли вызов какой-то функции к ошибке или нет, можно перед вызовом функции под подозрением очистить стек ошибок (используя цикл while пока `glGetError` не вернет 0) и после вызова функции вызвать `glGetError` и посмотреть, чему равно возвращаемое значение. Если не 0 — прозошла ошибка.

Второй вариант — `glDebugMessageCallback`. Этот вызов введен в OpenGL 4.3. Он позволяет получить детальную информацию в случае, если случилась какая-то ошибка. В качестве параметра этой функции передается указатель на коллбэк-функцию, которая будет вызвана, если произошла ошибка.

Сконцентрируемся пока на 1 варианте. Напишем функции для очистки стека ошибок и проверки на наличие ошибки:

```C++
static void GLClearError() {
	while (glGetError() != GL_NO_ERROR);
}

static void GLCheckError() {
	while (GLenum error = glGetError()) {
		std::cout << "[OpenGL Error] (" << error << ")" << std::endl;
	}
}
```

И теперь вызовем их соответственно до и после вызова подозрительной функции:

```C++
GLClearError();
glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr);
GLCheckError();
```

В консоль выведется сообщение об ошибке под номером 1280. В заголовочном файле glew.h определены все коды ошибок, но они там в 16-ричном коде. 1280 в 16-ричном коде равно 500, смотрим на ошибку под номером 500 и наши подозрения подтверждаются, это код ошибки GL_INVALID_ENUM.

Проблема с предложенным процессом отладки — придется перед каждым вызовом функций OpenGL очищать стек ошибок, потом после вызова выводить все ошибки. К тому же, как понять, какой именно вызов привел к ошибке, если мы проверяем каждый из них? Это можно решить, установив точку останова в теле самой функции GLCheckError и посмотреть, в каком месте она была вызвана.

Воспользуемся инструментом assertion чтобы автоматически определять строку, в которой произошла ошибка. В Visual Studio воспользуемся вызовом `__debugbreak();` для остановки выполняения программы, в g++ — вызовом `raise(SIGTRAP);`.

```C++
#define ASSERT(x) if (!(x)) __debugbreak();
```

Напишем функцию GLLogCall():

```C++
static bool GLLogCall() {
	while (GLenum error = glGetError()) {
		std::cout << "[OpenGL Error] (" << error << ")" << std::endl;
		return false;
	}
	return true;
}
```

```C++
GLClearError();
glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr);
ASSERT(GLogCall());
```

Теперь мы сможем отследить строку, в которой выполнение программы привело к ошибке.

Для того, чтобы не очищать стек ошибок каждый раз перед вызовом и не писать потом проверку на возникновение ошибки, напишем следующий макрос:

```C++
#define GLCall(x) GLClearError();\
	x;\
	ASSERT(GLLogCall())
```

 И теперь будем оборачивать каждый вызов из OpenGL в GLCall.

 Было бы неплохо еще вывести в консоль, вызов какой функции привел к ошибке, в каком файле и какой строке. Воспользуемся инструментами компилятора для этого. Сначала перемишем наш макрос GLCall:

 ```C++
#define GLCall(x) GLClearError();\
	x;\
	ASSERT(GLLogCall(#x, ))
```

И перепишем функцию проверки ошибки:

```C++
static bool GLLogCall(const char* function, const char* file, int line) {
	while (GLenum error = glGetError()) {
		std::cout << "[OpenGL Error] (" << error << ")" << std::endl;
		std::cout << "During execution of " << function << " in " << file << " at line " << line << std::endl;
		return false;
	}
	return true;
}
```

Теперь, если ошибка и произошла, мы сразу узнаем, при каком вызове, в каком файле и в какой строке. И результат нашей работы:

```C++
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

#define ASSERT(x) if (!(x)) __debugbreak();
#define GLCall(x) GLClearError();\
	x;\
	ASSERT(GLLogCall(#x, __FILE__, __LINE__))

static void GLClearError() {
	while (glGetError() != GL_NO_ERROR);
}

static bool GLLogCall(const char* function, const char* file, int line) {
	while (GLenum error = glGetError()) {
		std::cout << "[OpenGL Error] (" << error << ")" << std::endl;
		std::cout << "During execution of " << function << " in " << file << " at line " << line << std::endl;
		return false;
	}
	return true;
}

struct ShaderProgramSource {
	std::string VertexSource;
	std::string FragmentSource;
};

static ShaderProgramSource ParseShader(const std::string& filepath) {
	std::ifstream stream(filepath);

	enum class ShaderType {
		NONE = -1, VERTEX = 0, FRAGMENT = 1
	};

	std::stringstream ss[2];
	std::string line;
	ShaderType type = ShaderType::NONE;
	while (getline(stream, line)) {
		if (line.find("#shader") != std::string::npos) {
			if (line.find("vertex") != std::string::npos) {
				// здесь мы подгружаем vertex shader
				type = ShaderType::VERTEX;
			} else if (line.find("fragment") != std::string::npos) {
				// здесь мы подгружаем fragment shader
				type = ShaderType::FRAGMENT;
			}
		} else {
			ss[(int)type] << line << "\n";
		}
	}

	return { ss[0].str(), ss[1].str() };
}

static unsigned int CompileShader(unsigned int type, const std::string& source) {
	unsigned int id = glCreateShader(type);
	const char* src = source.c_str();
	glShaderSource(id, 1, &src, nullptr);
	glCompileShader(id);

	int result;
	glGetShaderiv(id, GL_COMPILE_STATUS, &result);
	if (result == GL_FALSE) {
		int length;
		glGetShaderiv(id, GL_INFO_LOG_LENGTH, &length);
		char* message = (char *) alloca(length * sizeof(char));
		glGetShaderInfoLog(id, length, &length, message);
		std::cout << "Failed to compile " << 
			(type == GL_VERTEX_SHADER ? "vertex" : "fragment") << " shader!" << std::endl;
		std::cout << message << std::endl;
		glDeleteShader(id);
		return 0;
	}

	return id;
}

static int CreateShader(const std::string& vertexShader, const std::string& fragmentShader) {
	unsigned int program = glCreateProgram();
	unsigned int vs = CompileShader(GL_VERTEX_SHADER, vertexShader);
	unsigned int fs = CompileShader(GL_FRAGMENT_SHADER, fragmentShader);
	glAttachShader(program, vs);
	glAttachShader(program, fs);
	glLinkProgram(program);
	glValidateProgram(program);

	glDeleteShader(vs);
	glDeleteShader(fs);

	return program;
}

int main(void)
{
	GLFWwindow* window;

	/* Initialize the library */
	if (!glfwInit())
		return -1;

	/* Create a windowed mode window and its OpenGL context */
	window = glfwCreateWindow(480, 480, "Hello World", NULL, NULL);
	if (!window) {
		glfwTerminate();
		return -1;
	}

	/* Make the window's context current */
	glfwMakeContextCurrent(window);

	if (glewInit() != GLEW_OK) {
		std::cout << "Error: couldn't connect GLEW" << std::endl;
	}

	float positions[] = {
		-0.5f,	-0.5f,
		 0.5f,	 -0.5f,
		 0.5f,	0.5f,
		 -0.5f,	0.5f,
	};

	unsigned int indices[] = {
		0, 1, 2,
		2, 3, 0,
	};

	unsigned int buffer;
	glGenBuffers(1, &buffer);
	glBindBuffer(GL_ARRAY_BUFFER, buffer);
	glBufferData(GL_ARRAY_BUFFER, 4 * 2 * sizeof(float), positions, GL_STATIC_DRAW);

	unsigned int ibo;
	glGenBuffers(1, &ibo);
	glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);
	glBufferData(GL_ELEMENT_ARRAY_BUFFER, 6 * sizeof(unsigned int), indices, GL_STATIC_DRAW);

	glEnableVertexAttribArray(0);
	glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), 0);

	ShaderProgramSource source = ParseShader("Basic.shader");
	std::cout << "VERTEX" << std::endl;
	std::cout << source.VertexSource << std::endl;
	std::cout << "FRAGMENT" << std::endl;
	std::cout << source.FragmentSource << std::endl;

	unsigned int shader = CreateShader(source.VertexSource, source.FragmentSource);
	glUseProgram(shader);

	/* Loop until the user closes the window */
	while (!glfwWindowShouldClose(window))
	{
		/* Render here */
		glClear(GL_COLOR_BUFFER_BIT);

B		GLCall(glDrawElements(GL_TRIANGLES, 6, GL_INT, nullptr));

		/* Swap front and back buffers */
		glfwSwapBuffers(window);

		/* Poll for and process events */
		glfwPollEvents();
	}

	glDeleteProgram(shader);
	glfwTerminate();
	return 0;
}
```
