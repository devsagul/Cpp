# Лекция 3

Темы, которые будут рассмотрены в данной лекции:

OpenGL;
GLFW;
Первая программа на OpenGl;
Использование современных функций OpenGL;
Vertex Buffers;
Shaders;


## OpenGL

OpenGL - API, позволяющий обращаться к ресурсам видеокарты. OpenGL не являюется какой-либо конкретной библиотектой, это спецификация. Имплементация OpenGL лежит в ответственности производителся вашей видеокарты. Как следствие, поведение той или иной функции может отличаться на видеокартах Nvidia и AMD. Сама имплементация может быть как открытой, так и закрытой.

При этом OpenGL в некотором смысле является кросс-платформенной. Один и тот же код может исполняться на разных платформах. Традиционно один и тот же движок использует несколько API в своей реализации для того, чтобы утилизировать возможности нативных для платформы API.

OpenGL является одним из самых простых в освоении инструментов.

Существует несколько библиотек, имплементирующих OpenGL:

- [GLFW](http://www.glfw.org)
- [GLUT](https://www.opengl.org)
- [FreeGLUT](http://freeglut.sourceforge.net/)
- [SDL](http://www.libsdl.org/)

## GLFW

GLFW - одна из самых легковесных библиотек для работы с OpenGL. Она позволит нам создать окно и получить доступ к контексту OpenGL.

Для использования GLFW можно как скачать исходный код и включить его в проект, так и скачать скомпилированные бинарники для Windows.

## Первая программа на OpenGL

Вот пример создания окна при помощи GLFW:

```C++
#include <GLFW/glfw3.h>

int main(void)
{
    GLFWwindow* window;

    /* Initialize the library */
    if (!glfwInit())
        return -1;

    /* Create a windowed mode window and its OpenGL context */
    window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        return -1;
    }

    /* Make the window's context current */
    glfwMakeContextCurrent(window);

    /* Loop until the user closes the window */
    while (!glfwWindowShouldClose(window))
    {
        /* Render here */
        glClear(GL_COLOR_BUFFER_BIT);

        /* Swap front and back buffers */
        glfwSwapBuffers(window);

        /* Poll for and process events */
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}
```

Нарисуем треугольник, добавив следующий код под glClear:

```C++
	glBegin(GL_TRIANGLES);
	glVertex2f(-0.5f, -0.5f);
	glVertex2f(0.0f, 0.5f);
	glVertex2f(-0.5f, 0.5f);
	glEnd;
```

Таким образом, мы создали кросплатформенное приложение, которое загружает окно и внутри него позволяет отрендерить изображение (треугольник, в данном случае).

## Использование современных функций OpenGL

Для использования современных функций OpenGL нам понадобится какая-то библиотека, которая сможет обращаться к драйверам видеокарты. Из вариантов - GLEW и GLUT. В данном случае мы воспользуемся GLEW как более простым в освоении вариантом.

Для того, чтобы использовать функции OpenGL необходимо вызвать glewInit при созданном конексте. При статическом связывании, необходимо определить макрос `GLUT_STATIC` и затем после создания контекста `glfwMakeContextCurrent(window);` вызвать функцию `glewInit()`.

С этого момента у вас есть доступ ко всем функциям OpenGL.

Для проверки правильности иницаилизации GLEW, необходимо проверить значение, которое возвращает вызов `glewInit()`. Для индикации корректного запуска определена макроподстановка `GLEW_OK`. Любое другое значение сигнализирует об ошибке.

## Vertex Buffers

Vertex Buffer Objects - способ выгрузки данных в память видеокарты. Если говорить простым языком, то, чтобы использовать ресурсы видеокарты, нам придется определить способ хранения данных и способ их преобразования. В данном случае Vertex Buffers - память.

За пределами нашего основного цикла взаимодействия определим буфер:

```C++
float positions[6] = {
	-0.5f, -0.5f,
	 0.0f, 0.5f,
	 0.5f, -0.5f
};

unsigned int buffer;
glGenBufffers(1, &buffer);
glBindBuffer(GL_ARRAY_BUFFER, bufffer);
glBufferData(GL_ARRAY_BUFFER, 6 * sizeof(float), positions, GL_STATIC_DRAW);
```

И для рендера нашего треугольника будем использовать вызов:

```C++
glDrawArrays(GL_TRIANGLES, 0, 3);
```

Где positions - обычный массив из чисел с плавающей точкой, который хранится в оперативной памяти, buffer - индекс определяемого буфера, glGenBuffers - вызов для генерации заданного количества буферов с заданным индексом, glBindBuffer - указание типа буфера, glBufferData - загрузка данных в буфер.

Пока что запуск нашей программы не приводит к успешному отображению, нам необходимо определить то, как интерпретировать данный в буфере и написать shader, который и будет рендерить треугольник.

Для того, чтобы иметь возможность интерпретировать данный корректно из шейдера, необходимо указать структуру атрибутов в буфере. В нашем случае понадобятся следующие два вызова:

```C++
glEnableVertexAttrgibArray(0);
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), 0);
```

Сначала мы указали, что в нашем буфере есть аттрибут с индексом 0, а затем определили необходимы данные для этого аттрибута:

индекс, количество данных (в единицах типа данных, например, GL_FLOAT), тип данных, нормализация, шаг, смещение. Шаг указывает, насколько далеко находятся два элемента в буфере, т.е. сколько памяти занимает каждый элемент, а смещение указывает, где в каждой структуре располагается данный аттрибут.

## Shaders

Шейдеры - вторая часть набора для работы с ресурсами видеокарты. По большому счету, это функции, которые будут выполнены на чипе над теми данными, которые загружены в буферы.

```C++
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <iostream>

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
	if (!window)
	{
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
	unsigned int shader = CreateShader(vertexShader, fragmentShader);
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
