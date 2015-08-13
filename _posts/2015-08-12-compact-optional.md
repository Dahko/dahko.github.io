---
layout: post
title: Compact optional
---
Наткнулся на&nbsp;любопытную [статью про эффективные optional](https://akrzemi1.wordpress.com/2015/07/15/efficient-optional-values/) в&nbsp;Andrzej&rsquo;s C++&nbsp;blog.

`optional`&nbsp;&mdash; это переменная, в&nbsp;которой может быть валидное значение, а&nbsp;может не&nbsp;быть. В&nbsp;Haskell и&nbsp;иже с&nbsp;ним такие штуки описываются монадой `Maybe`, а&nbsp;в&nbsp;С++ есть реализация `Boost::optional`, как и&nbsp;миллионы доморощенных реализаций&nbsp;&mdash; благо, класс несложный. 

Простейшая реализация выглядит примерно так:

<!--more-->

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

В&nbsp;классе кроме собственно значения элемента хранится `bool`. Это приводит к&nbsp;дополнительному расходу памяти, вплоть до&nbsp;двукратного. Например, в&nbsp;реализации Boost `bool` идет _перед_ значением, которое к&nbsp;тому&nbsp;же выравнивается в&nbsp;памяти по&nbsp;своему размеру, т.е. как раз двукратное превышение.

Соответственно и&nbsp;по&nbsp;памяти это дело распухает, и&nbsp;в&nbsp;кэш хуже ложится. Другой издревле известный способ сигнализировать о&nbsp;том, что переменная пуста&nbsp;&mdash; использование магических значений типа &minus;1. Т.е.

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

Такая магия идеальна по&nbsp;производительности, но&nbsp;хромает интерфейс: пользователь должен помнить, какое значение является магическим (-1, 0, `MAX_INT`, ...), а&nbsp;также может забыть сделать проверку.

Автор предлагает объединить оба эти подхода, воспользовавшись шаблонами. `compact_optional` будет иметь второй шаблонный аргумент, куда принимает policy&nbsp;&mdash; структурку со&nbsp;статическими функциями, умеющую возвращать магическое значение и&nbsp;проверять переданное число на&nbsp;равенство магическому значению. Примерно так:
{% highlight cpp %}
struct my_int_policy
{
	static int empty_value() { return -1; }
	static bool is_empty_value(const int& v) { return v == -1; }
};
{% endhighlight %}

Тогда в&nbsp;самом `compact_optional` хранится только значение нужного типа, а&nbsp;индикатором валидности служит не `bool`, а&nbsp;наше магическое значение (в&nbsp;данном случае &minus;1).

Использование&nbsp;&mdash; как у&nbsp;обычного `optional`, с&nbsp;теми&nbsp;же функциями `IsValid()` и `GetValue()`. Недостатки&nbsp;&mdash; надо указывать policy и&nbsp;одно значение в&nbsp;типе жертвуется под магическое. 

В&nbsp;реальной жизни очень часто по&nbsp;смыслу переменная использует не&nbsp;весь доступный диапазон значений (например не&nbsp;может быть 2&nbsp;147&nbsp;483&nbsp;647 яблок или элементов в&nbsp;массиве), так что пожертвовать всегда есть что.

Думаю, эта идея&nbsp;&mdash; совместить плюсы четкого интерфейса и&nbsp;эффективность магических значений&nbsp;&mdash; может пригодиться не&nbsp;только в&nbsp;C++, но&nbsp;и&nbsp;в&nbsp;других языках програмирования, просто реализация будет немного другой.

Подробности и&nbsp;более полное описание плюсов и&nbsp;минусов&nbsp;&mdash; [в&nbsp;оригинальной статье](https://akrzemi1.wordpress.com/2015/07/15/efficient-optional-values/).