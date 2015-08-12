---
layout: post
title: Compact optional
---
Наткнулся на любопытную [статью про эффективные optional](https://akrzemi1.wordpress.com/2015/07/15/efficient-optional-values/) в Andrzej's C++ blog.

Optional &mdash; это переменная, в которой может быть валидное значение, а может не быть. В Haskell и иже с ним такие штуки описываются монадой `Maybe`, а в С++ есть реализация `Boost::optional`, как и миллионы доморощенных реализаций &mdash; благо, класс несложный. 

Простейшая реализация выглядит примерно так:

{% highlight cpp %}
template<typename T>
class optional
{
private:
	T val;
	bool initialized;
public:
	optional() : initialized(false) {}
	optional( T const& _val ) : initialized(true), val(_val) {}
	optional& operator=( T const& _val ) 
		{ initialized=true; val=_val; return *this; }
	
	bool IsValid() { return initialized; }
	T const& GetValue() { return val; }
};

optional<float> SquareRoot( float arg )
{
	if ( arg > 0.0f )
		return sqrt(arg);
	else
		return optonal<float>();
	
}

void main()
{
	float val = -10.0f;
	optional<float> v1 = SquareRoot( val );
	if ( v1.IsValid() )
		printf("Root of %f is %f", val, v1.GetValue() );
	else
		printf("Warning! Root of %f is imaginary", val);
}
{% endhighlight %}

В реализации кроме собственно значения элемента хранится `bool`. Это приводит к дополнительному расходу памяти, вплоть до двукратного. Например, в реализации Boost `bool` идет перед значением, которое к тому же выравнивается в памяти по своему размеру, т.е. как раз двукратное превышение.

Соответственно и по памяти это дело распухает, и в кэш хуже ложится. Другой издревле известный способ сигнализировать о том, что переменная пуста &mdash; использование магических значений типа -1. Т.е.

{% highlight cpp %}
float SquareRoot( float arg )
{
	if ( arg > 0.0f )
		return sqrt(arg);
	else
		return -1.0f;
}
// В main вместо IsValid проверяем на равенство -1 
// (вопросы сравнения float'ов тут опустим).
{% endhighlight %}

Такая магия идеальна по производительности, но хромает интерфейс: пользователь должен помнить, какое значение является магическим (-1, 0, `MAX_INT`, ...), а также может забыть сделать проверку.

Автор предлагает объединить оба эти подхода, воспользовавшись шаблонами. `compact_optional` будет иметь второй шаблонный аргумент, куда принимает policy &mdash; структурку со статическими функциями, умеющую возвращать магическое значение и проверять переданное число на равенство магическому значению. Примерно такое:
{% highlight cpp %}
struct my_int_policy
{
	static int empty_value() { return -1; }
	static bool is_empty_value(const int& v) { return v == -1; }
};
{% endhighlight %}

Тогда в самом `compact_optional` хранится только значение нужного типа, а индикатором валидности служит не `bool`, а наше магическое значение (в данном случае -1).

Использование &mdash; как у обычного `optional`, с теми же функциями `IsValid()` и `GetValue()`. Недостатки &mdash; надо указывать policy и одно значение в типе жертвуется под магическое. 

В реальной жизни очень часто по смыслу переменная использует не весь диапазон значений int'а (например не может быть 2,147,483,647 яблок или элементов в массиве), так что пожертвовать всегда есть что.

Думаю, эта идея &mdash; совместить плюсы четкого интерфейса и эффективность магических значений &mdash; может пригодиться не только в C++, но и в других языках програмирования, просто реализация будет немного другой.

Подробности и более полное описание плюсов и минусов &mdash; [в оригинальной статье](https://akrzemi1.wordpress.com/2015/07/15/efficient-optional-values/).