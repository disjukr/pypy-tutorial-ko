PyPy로 인터프리터 작성하기
==========================
이 글은 Andrew Brown <brownan@gmail.com>에 의해 작성되었으며,
pypy-dev 메일링 리스트에 있는 개발자들의 도움을 받았습니다.

이 튜토리얼의 원본과 주변 파일들은
https://bitbucket.org/brownan/pypy-tutorial/ 에서 구할 수 있습니다.

제가 처음 PyPy 프로젝트를 알게 되었을 때는 이게 대체 무엇인지 깨닫는데 시간이 꽤
걸렸습니다. 저와 같은 사람들을 위해 설명하자면 PyPy는 다음의 두 부분으로 요약할
수 있습니다:

 * 인터프리터 구현을 위한 도구의 집합
 * 그 것을 사용하여 만들어진 파이썬 구현체

두번째 부분은 아마 대부분의 사람들이 생각하는 PyPy일 것입니다.
하지만 이 튜토리얼에서는 그 파이썬 인터프리터에 대해서는 다루지 *않습니다*.
이 튜토리얼은 자신의 언어를 위한 인터프리터를 직접 구현하는 법에 대한
내용입니다.

이는 저 스스로 PyPy가 정확히 무엇인지, 어떻게 동작하는지를 이해하기 위해 진행한
프로젝트이기도 합니다.

이 튜토리얼은 독자가 PyPy를, 그것이 어떻게 돌아가는지를, 심지어 그게 무엇인지를
잘 모를 수 있을 것이라 가정합니다. 저는 처음부터 차근차근 설명할 요량입니다.

PyPy는 무엇을 하는가
--------------------
PyPy가 무엇을 할 수 있는지 대략적인 개요를 설명하겠습니다.
일단 여러분이 인터프리터를 하나 만들고 싶다고 칩시다.
그 인터프리터는 소스코드 파서와, 바이트코드 해석 루프,
그 외에 잡다한 표준 라이브러리 코드를 포함하는 거구요.

적당히 복잡한 언어라면 작업량이 꽤나 되겠죠. 그리고 상당한 로우레벨 작업이
수반됩니다. 파서와 컴파일러 코드를 작업하는건 대체로 재미가 없는 작업이라,
세상에는 파서와 컴파일러를 자동으로 생성해주는 도구가 존재합니다.

그럼에도 인터프리터를 구현하려면 메모리 관리를 고려해야 하고, 임의의 정밀도를
가진 정수나 멋지게 일반화된 해시테이블 등등을 원한다면 바퀴를 다시 만드는 노력이
필요합니다.
이는 누군가 언어를 하나 만들고 싶다는 생각을 무르게 만드는데 충분한 요소입니다.

파이썬같이, 이미 존재하는 고수준 언어 위에서 자신의 언어를 작성할 수 있다면
멋지겠죠? 자동화된 메모리 관리나 풍부한 자료구조 등 고수준 언어에 구현된 기능을
그대로 가져다 사용할 수 있을겁니다.

엇, 근데 인터프리터 위에서 돌아가는 인터프리터라면 많이 느리지 않을까요?
그렇게 만들어버린다면 인터프리팅이 두 번에 걸쳐서 일어날 텐데요.

눈치 채셨겠지만 PyPy는 이 문제를 해결합니다. PyPy는 여러분의 인터프리터 코드를
분석 및 C 코드(또는 JVM이나 CLI)로 변환하기 위한 정교한 툴체인입니다.
이 툴체인은 "변환(translation)"을 수행하기 위해, 어떻게 많은 파이썬 문법과
표준 라이브러리들을 옮겨내야 하는지를 알고 있습니다. 여러분이 작업해야할 것은
여러분의 인터프리터를 파이썬의 서브셋인 **RPython**으로 작성하는 것 뿐입니다.
RPython은 분석과 변환 작업 및 매우 효율적인 인터프리터를 생성할 수 있도록
매우 조심스럽게 정의된 언어입니다.

효율적인 인터프리터는 작성하기 쉬워야 하니까요.

언어
----
구현하기에 굉장히 쉬운 언어를 하나 골랐습니다. 런타임은 모두 `0`으로 초기화된
정수 테이프와, 테이프 중 한 칸을 가르키는 포인터 하나로 구성되어있는 언어인데,
이 언어는 8가지 명령어로 구성되어있습니다:

### >
테이프 포인터를 오른쪽으로 한 칸 이동시킵니다.

### <
테이프 포인터를 왼쪽으로 한 칸 이동시킵니다.

### +
포인터가 가르키는 칸의 값을 증가시킵니다.

### -
포인터가 가르키는 칸의 값을 감소시킵니다.

### [
만약 포인터가 가르키는 칸의 값이 `0`이라면 상응하는 괄호짝 `]` 뒤로 건너뜁니다.

### ]
상응하는 괄호짝 `[`로 돌아갑니다 (반복 조건도 평가합니다).

### .
포인터가 가르키는 칸의 값을 `stdout`으로 한 바이트 출력합니다.

### ,
`stdin`에서 한 바이트 읽어와 포인터가 가르키는 칸에 대입합니다.

인식되지 않는 바이트들은 무시합니다.

이게 무엇인지 아는 분들이 있으시겠죠. 전 앞으로 이 것을 BF라고 부르겠습니다.

한가지 주목할 점은 이 언어는 바이트코드가 바로 언어 자신이라는 점입니다;
달리 소스코드를 바이트코드로 변환할 필요가 없습니다.
따라서 이 언어는 읽는 즉시 해석할 수 있습니다:
우리 인터프리터의 메인 평가 루프는 소스코드에 작성된 그대로 돌아갈 겁니다.
이런건 구현이 매우 단순하게 됩니다.

첫 걸음
-------
일반적인 파이썬 문법으로 BF 인터프리터를 작성하는 것부터 시작해봅시다.
가장 먼저, 평가 루프를 작성합니다:

```python
def mainloop(program):
    tape = Tape()
    pc = 0
    while pc < len(program):
        code = program[pc]

        if code == ">":
            tape.advance()
        elif code == "<":
            tape.devance()
        elif code == "+":
            tape.inc()
        elif code == "-":
            tape.dec()
        elif code == ".":
            sys.stdout.write(chr(tape.get()))
        elif code == ",":
            tape.set(ord(sys.stdin.read(1)))
        elif code == "[" and value() == 0:
            # Skip forward to the matching ]
        elif code == "]" and value() != 0:
            # Skip back to the matching [

        pc += 1
```

프로그램 카운터(pc)는 현재 명령어의 인덱스를 잡고 있습니다.
반복문 안쪽의 첫번째 문장에서는 pc가 가르키는 위치에서 명령을 읽어오고,
그 다음 복합 if 문에선 읽어온 명령을 어떻게 처리할지를 결정합니다.

여기에서는 `[`와 `]`의 구현이 빠져있는데, 그 두 개의 명령어는 프로그램 카운터의
값을 일치하는 짝의 위치로 변경시킬 수 있습니다. (그 후에 `pc`값은 증가하게 되며,
따라서 반복 조건은 루프의 진입시에 한 번 평가되고, 그 뒤로는 매 주기의 끝마다
평가됩니다)

다음은 테이프 자신의 내용과 그 것을 가르키는 포인터를 들고있는
`Tape` 클래스의 구현입니다:

```python
class Tape(object):
    def __init__(self):
        self.thetape = [0]
        self.position = 0

    def get(self):
        return self.thetape[self.position]
    def set(self, val):
        self.thetape[self.position] = val
    def inc(self):
        self.thetape[self.position] += 1
    def dec(self):
        self.thetape[self.position] -= 1
    def advance(self):
        self.position += 1
        if len(self.thetape) <= self.position:
            self.thetape.append(0)
    def devance(self):
        self.position -= 1
```

테이프는 오른쪽으로 필요한 만큼 무한정 확장됩니다.
포인터가 음수가 되지 않도록 에러 체크를 해줘야 하겠지만
저는 일단 그 부분은 걱정하지 않도록 하겠습니다.

생략한 `[`와 `]` 구현을 제외하면 이 코드는 제대로 돌아갈 것입니다.
하지만 프로그램이 주석을 많이 갖고있다면,
쓸데없이 런타임에 그 것들을 넘어가는 처리가 일어날 테지요.
그러니 처음에 한 번 파싱해서 걸러내어 버리도록 합시다.

하는 김에 괄호간에 딕셔너리 매핑을 만들어서, 다음과 같이 짝이 맞는 괄호를
한 번의 딕셔너리 룩업으로 찾을 수 있게 합시다:

```python
def parse(program):
    parsed = []
    bracket_map = {}
    leftstack = []

    pc = 0
    for char in program:
        if char in ('[', ']', '<', '>', '+', '-', ',', '.'):
            parsed.append(char)

            if char == '[':
                leftstack.append(pc)
            elif char == ']':
                left = leftstack.pop()
                right = pc
                bracket_map[left] = right
                bracket_map[right] = left
            pc += 1

    return "".join(parsed), bracket_map
```

이 함수는 모든 불필요한 명령어를 제거한 문자열과, 괄호 인덱스가 짝이 맞는 괄호의
인덱스로 매핑되는 딕셔너리를 반환합니다.

이제 남은 코드들을 이어붙이면 제대로 작동하는 BF 인터프리터가 됩니다:

```python
def run(input):
    program, map = parse(input.read())
    mainloop(program, map)

if __name__ == "__main__":
    import sys
    run(open(sys.argv[1], 'r'))
```

처음부터 따라오셨다면 `mainloop()`에서 인자를 하나 더 받고 if 문에서 괄호를
처리하는 부분을 구현하시면 됩니다. 완전한 예제는 다음 파일을 참고하면 됩니다:
[example1.py](./example1.py)

지금 바로 인터프리터가 제대로 작동하는지 보기 위해 파이썬으로 돌려볼 수는
있습니다만, 좀 더 복잡한 예제파일을 돌리기에는 *매우* 느리게 작동할 것입니다:

```sh
$ python example1.py 99bottles.b
```

이 튜토리얼의 저장소에서 `mandel.b`나 다른 예제 프로그램들(제가 작성한 것은
아닙니다)을 찾아서 돌려보세요.

PyPy 변환
---------
근데 원래 목적은 PyPy를 설명하기 위함이지 BF 인터프리터 튜토리얼이 아니죠.
여기에 PyPy를 끼얹어서 짱짱-빠른 실행파일로 만들려면 이제 뭘 해야 할까요?

잠깐 다른 이야기를 하자면 PyPy 소스 트리에서 pypy/translator/goal 디렉토리에
들어있는 예제들이 도움이 많이 됩니다. 저는 PyPy의 헬로월드 격인
"targetnopstandalone.py" 예제를 보는 것으로 시작했답니다.

다시 예제로 돌아와서, 모듈은 프로그램의 진입 지점을 반환하는 "target"을 정의해야
합니다. 변환 프로세스는 모듈을 불러와서 "target" 함수를 실행하여 변환을 시작할
함수 객체를 얻어냅니다:

```python
def run(fp):
    program_contents = ""
    while True:
        read = os.read(fp, 4096)
        if len(read) == 0:
            break
        program_contents += read
    os.close(fp)
    program, bm = parse(program_contents)
    mainloop(program, bm)

def entry_point(argv):
    try:
        filename = argv[1]
    except IndexError:
        print "You must supply a filename"
        return 1

    run(os.open(filename, os.O_RDONLY, 0777))
    return 0

def target(*args):
    return entry_point, None

if __name__ == "__main__":
    entry_point(sys.argv)
```

변환후에 만들어진 실행파일을 돌리면 `entry_point` 함수로 명령행 인자들이
전달됩니다.
바뀐 사항들이 몇 가지 더 있는데요, 다음 섹션을 봅시다...

RPython에 대해서
----------------
지금은 잠깐 RPython을 짚고 넘어가겠습니다. PyPy는 아무 파이썬 코드나 다
번역할 수 있는 도구가 아닌데, 왜냐하면 파이썬이 너무 동적이기 때문입니다.
그래서 사용할 수 있는 표준 라이브러리 함수나 문법 구조 등에 몇 가지 제약사항이
있습니다. 제가 그 제약사항을 일일이 짚고 넘어가진 않을 거구요, 자세한 내용은
[이 문서](https://pypy.readthedocs.org/en/latest/coding-guide.html#id1)를
보시면 됩니다.

위의 예제에서 아마 몇가지 바뀐 사항들을 확인할 수 있을 겁니다. 전 이제
파일 객체를 사용하는 대신 저수준의 `os.open`과 `os.read`로 file descriptor를
사용하고 있습니다.
위에 써놓진 않았지만 `.`과 `,`의 구현도 비슷하게 변형되어 있습니다.
이 정도만 맞춰두면 PyPy를 소화해내기 위한 나머지 작업은 단순한 편입니다.

그렇게 어렵진 않았죠? 전 아직도 딕셔너리를 사용하고 있고, 길이가 가변적인
리스트를 사용하는 데다 클래스랑 객체도 활용하고 있답니다! 그리고 지금 사용하는
file descriptor마저 너무 저수준이라 문제라면, PyPy의 "Rpython 표준 라이브러리"에
포함되어있는 `rlib.streamio`라는 대안도 있습니다.

지금까지 다룬 내용이 적용된 예제파일은 여기 있습니다:
[example2.py](./example2.py)

변환
----
bitbucket.org 저장소에 있는 PyPy 최신 버전을 준비합니다:

```sh
$ hg clone https://bitbucket.org/pypy/pypy
```

(제 예제가 돌아가려면 최신 버전에 있는 버그 수정이 반영되어야 해서 그렇습니다.)

우리가 돌릴 스크립트는 `pypy/translator/goal/translate.py` 경로에 있습니다.
이 스크립트에 예제 모듈을 인자로 넣고 돌려봅시다:

```sh
$ python ./pypy/rpython/bin/rpython example2.py
```

(속도를 더 내려면 PyPy의 파이썬 인터프리터를 사용해도 되는데,
꼭 필요한건 아닙니다.)

PyPy가 바쁘게 돌아가는 동안 심심하지 않게 콘솔에 이쁜 프랙탈이 그려집니다.
제 머신에선 20초 정도 걸리네요.

다 끝나면 BF 프로그램을 돌릴 수 있는 바이너리가 나옵니다.
이 튜토리얼 저장소엔 제 컴퓨터에서 45초 정도 걸리는 만델브로트 프랙탈 생성기 등
몇가지 BF 프로그램들이 들어있는데, 그 인터프리터로 한 번 돌려보세요:

```sh
$ ./example2-c mandel.b
```

변환을 안 거치고 그대로 파이썬에서 돌릴 경우랑 비교해봅시다:

```sh
$ python example2.py mandel.b
```

안끝나죠, 그죠?

다 했어요. RPython으로 작성한 인터프리터를
PyPy 툴체인으로 별 탈 없이 번역하는데 성공했습니다.

JIT 끼얹기
----------
RPython을 C로 변환한 것 까지는 좋은데 PyPy를 사용하면서
*just-in-time 컴파일러를 날로먹는 기능*을 안쓰고 넘어가긴 아깝죠.
네, 인터프리터를 어떻게 구성했는지에 대해서 몇가지 힌트만 던져주면
PyPy가 알아서 런타임에, 해석된 BF 언어를 기계어로 바꿔주는
JIT 컴파일러를 만들어서, 여러분의 인터프리터에 끼워줍니다!

그래서 PyPy한테 그 일을 부탁하려면 뭘 알려줘야 할까요?
일단 PyPy는 타겟 언어에서(BF) 명령어를 실행한 과정을 쟁여놓고 분석하기 때문에
여러분의 바이트코드 평가 루프가 어디서 시작하는지를 알아야 합니다.

그리고 또 알려줘야 하는게 특정 실행 프레임을 정의하는 것이 무엇인지 인데,
BF는 딱히 스택 프레임을 갖지 않아서, 특정 명령어를 실행하기 위한 상수가 무엇인지,
그리고 그게 아닌 상수는 무엇인지 정도로 졸아듭니다.
이 것들은 각각 "초록"과 "빨강" 변수라고 불립니다.

잠깐 [example2.py](./example2.py)를 다시 보고 넘어갑시다.

`mainloop`를 보면 `pc`, `program`, `bracket_map`, `tape`
이렇게 네 가지 변수가 사용되고 있습니다.
여기서 `pc`, `program`, `bracket_map`은 초록 변수들입니다.
그 것들은 특정 명령어의 실행을 *정의*하고 있죠.
만약 JIT 루틴이 같은 조합의 초록 변수들을 먼저 발견한다면 그냥 무시하고 건너 뛸지
반복 구문을 실행해야 할지를 알 수 있게 됩니다.
`tape` 변수는 코드 실행에 의해 조작되는 빨강 변수입니다.

이걸 PyPy한테 알려줍시다.
`JitDriver` 클래스를 불러오고 인스턴스를 만드는 것부터 시작합니다:

```python
from rpython.rlib.jit import JitDriver
jitdriver = JitDriver(greens=['pc', 'program', 'bracket_map'],
        reds=['tape'])
```

그리고 `mainloop` 함수에서 반복문의 제일 상단에 이 코드를 추가합니다:

```python
jitdriver.jit_merge_point(pc=pc, tape=tape, program=program,
        bracket_map=bracket_map)
```

그리고 `JitPolicy`도 정의해줘야 합니다. 딱히 대단한 걸 하려는 게 아니므로
그냥 파일의 아무 곳에 다음과 같이 써주기만 하면 됩니다:

```python
def jitpolicy(driver):
    from rpython.jit.codewriter.policy import JitPolicy
    return JitPolicy()
```

예제 링크입니다: [example3.py](./example3.py)

이제 `--opt=jit` 플래그를 붙여서 다시 번역시켜봅시다:

```sh
$ python ./pypy/rpython/bin/rpython --opt=jit example3.py
```

JIT을 활성화시키면 변환이 엄청 오래걸리고 결과로 뽑히는 바이너리도 훨씬 큽니다.
제 머신에선 다 번역되는데 거의 8분이 걸렸네요.
다 끝나면 예의 만델브로트 프로그램을 다시 돌려봅시다.
번역이 엄청 오래걸린 반면, 프로그램 실행은 45초나 걸리던 전과는 달리
이번엔 12 초만에 처리됩니다!

잘 보면 흥미롭게도 JIT 컴파일러가,
만델브로트 예제가 해석된 코드를 기계어로 바뀌어나가는 순간을 발견할 수 있을겁니다.
처음 몇 줄이 출력되고 난 뒤부터 프로그램이 점점 더 속도를 높여나갑니다.

Tracing JIT 컴파일러에 대해서
-----------------------------
Tracing JIT 컴파일러가 어떻게 돌아가는지 지금 짚고 넘어가면 좋을 것 같습니다.
간략하게 설명하자면..

기본적으로는 여러분이 작성한 코드대로 돌아가는 인터프리터가 하나 있습니다.
그 인터프리터가 타겟 언어(BF)에서 반복되는 코드를 발견하면 해당 루프를
"핫"한 지점으로 간주하고 다음부터는 추적되도록(traced) 마킹합니다.
이후에 해당 루프에 진입하게 되면
그 인터프리터는 모든 실행 명령어를 로깅하는 추적 모드로 돌아갑니다.

루프가 끝나면 추적도 멈춥니다. 추적한 내용은 최적화 공정으로 던져지고,
어셈블러를 거쳐 기계어로 나오게 됩니다.
그 기계어는 이후에 해당하는 반복 구문을 돌 때 사용됩니다.

이 기계어는 대부분의 경우에 대해서 잘 돌아가도록 최적화 되어있지만,
해석된 코드에 대해서 몇가지 가정을 둡니다.
따라서 기계어 코드는 이 가정을 확인하기 위한 가드(guards)를 갖고 있는데요,
만약에 가드 체크가 실패하게 된다면 런타임은
본래의 해석 모드로 돌아가게(falls back) 됩니다.

JIT 컴파일러에 대해서는 다음의 링크에서 더 자세히 알아볼 수 있습니다:
[http://en.wikipedia.org/wiki/Just-in-time_compilation](http://en.wikipedia.org/wiki/Just-in-time_compilation)

디버깅과 로그 추적
------------------
여기서 더 할만한 게 있을까요? 심심한데 JIT이 뭘 하고 있는지 구경이나 해볼까요?

일단, 디버그 추적 로깅에 사용되는 `get_printable_location` 함수를 작성합시다:

```python
def get_location(pc, program, bracket_map):
    return "%s_%s_%s" % (
            program[:pc], program[pc], program[pc+1:]
            )
jitdriver = JitDriver(greens=['pc', 'program', 'bracket_map'], reds=['tape'],
        get_printable_location=get_location)
```

이 함수는 초록 변수들을 인자로 받아서 문자열을 반환하는 함수입니다.
이걸로 BF 코드를 밑줄로 구분해서 현재 실행하는 명령어를 볼 수 있게 출력해보죠.

이걸 [example4.py](./example4.py)로 다운받아서 example3.py 때랑 똑같이
번역해봅시다.

번역이 끝나면 테스트 프로그램(test.b 파일이고, 문자 "A"를 15번 정도
반복해서 출력하는 단순한 프로그램입니다.)을 추적 로깅하면서 돌려봅시다:

```sh
$ PYPYLOG=jit-log-opt:logfile ./example4-c test.b
```

프로그램을 돌리고 나서 "logfile" 파일을 열어봅시다.
이 파일은 꽤 읽기 어렵게 생겼군요, 이 참에 제가 설명을 좀 해보겠습니다.

이 파일은 수행된 모든 추적에 대한 로그를 담고 있어서,
여기서 무슨 명령어가 기계어로 번역되고 있는지 흘겨볼 수 있습니다.
혹시나 최적화 할만한 위치나 불필요한 명령어가 있는지도 찾아볼 수 있죠.

각각의 추적 로그는 다음과 같은 모양의 라인으로 시작합니다:

```
[3c091099e7a4a7] {jit-log-opt-loop
```

그리고 이런 모양으로 끝납니다:

```
[3c091099eae17d jit-log-opt-loop}
```

그 다음 줄은 몇 번 루프인지, 몇 개의 op들로 구성됐는지를 알려줍니다.
제 경우에 첫번째 추적 로그는 다음과 같이 생겼습니다:

```
1  [3c167c92b9118f] {jit-log-opt-loop
2  # Loop 0 : loop with 26 ops
3  [p0, p1, i2, i3]
4  debug_merge_point('+<[>[_>_+<-]>.[<+>-]<<-]++++++++++.', 0)
5  debug_merge_point('+<[>[>_+_<-]>.[<+>-]<<-]++++++++++.', 0)
6  i4 = getarrayitem_gc(p1, i2, descr=<SignedArrayDescr>)
7  i6 = int_add(i4, 1)
8  setarrayitem_gc(p1, i2, i6, descr=<SignedArrayDescr>)
9  debug_merge_point('+<[>[>+_<_-]>.[<+>-]<<-]++++++++++.', 0)
10 debug_merge_point('+<[>[>+<_-_]>.[<+>-]<<-]++++++++++.', 0)
11 i7 = getarrayitem_gc(p1, i3, descr=<SignedArrayDescr>)
12 i9 = int_sub(i7, 1)
13 setarrayitem_gc(p1, i3, i9, descr=<SignedArrayDescr>)
14 debug_merge_point('+<[>[>+<-_]_>.[<+>-]<<-]++++++++++.', 0)
15 i10 = int_is_true(i9)
16 guard_true(i10, descr=<Guard2>) [p0]
17 i14 = call(ConstClass(ll_dict_lookup__dicttablePtr_Signed_Signed), ConstPtr(ptr12), 90, 90, descr=<SignedCallDescr>)
18 guard_no_exception(, descr=<Guard3>) [i14, p0]
19 i16 = int_and(i14, -9223372036854775808)
20 i17 = int_is_true(i16)
21 guard_false(i17, descr=<Guard4>) [i14, p0]
22 i19 = call(ConstClass(ll_get_value__dicttablePtr_Signed), ConstPtr(ptr12), i14, descr=<SignedCallDescr>)
23 guard_no_exception(, descr=<Guard5>) [i19, p0]
24 i21 = int_add(i19, 1)
25 i23 = int_lt(i21, 114)
26 guard_true(i23, descr=<Guard6>) [i21, p0]
27 guard_value(i21, 86, descr=<Guard7>) [i21, p0]
28 debug_merge_point('+<[>[_>_+<-]>.[<+>-]<<-]++++++++++.', 0)
29 jump(p0, p1, i2, i3, descr=<Loop0>)
30 [3c167c92bc6a15] jit-log-opt-loop}
```

`debug_merge_point` 라인들은 너무 길어서 좀 잘라냈으니 양해해주세요.

자 이게 무엇을 하는지 좀 봅시다. 네 개의 인자—두 개의 객체 포인터(`p0`, `p1`)와
두 개의 정수(`i2`, `i3`)를 받아서 추적하네요.
디버그 라인들을 보면 `[>+<-]` 반복의 한 주기를 추적하는 것 같습니다.

4번째 줄의 `>` 부터 추적이 되는데 아무 것도 안하고 즉시 다음 명령으로 넘어갑니다.
`>`는 아무런 명령어도 들어가지 않고 완벽하게 최적화되어 빠져나간 것 같습니다.
이 루프는 언제나 테이프의 같은 부분에서만 돌아가야 하기 때문에
이 추적에서 테이프 포인터는 상수가 됩니다.
이 처리를 위한 명령을 명시적으로 넣어줄 필요가 없죠.

5 ~ 8번 줄에서는 `+` 명령을 위한 처리를 합니다.
먼저 포인터 `p1`이 가르키는 배열의 항목을 `i2`로 가져와서 (6번 줄), `1`을 더해서
`i6`에 저장하고 (7번 줄), 그걸 다시 배열에다 저장합니다 (8번 줄).

9번 줄은 `<` 명령으로 시작하지만 이 것 역시 아무런 명령어도 들어가지 않습니다.
보아하니 `i2`와 `i3`에 이 루틴의 두 테이프 포인터가 미리 계산되어 전달되는 것 같죠.
아직 `p0`이 어떤 녀석인지는 잘 모르겠지만 `p1`은 테이프 배열인 것 같습니다.

10번부터 13번 줄은 `-` 명령을 수행합니다: 배열 값을 읽어와서 (11번 줄),
`1`을 빼고 (12번 줄) 배열에 값을 설정합니다 (13번 줄).

Next, on line 14, we come to the "]" operation. Lines 15 and 16 check whether
i9 is true (non-zero). Looking up, i9 is the array value that we just
decremented and stored, now being checked as the loop condition, as expected
(remember the definition of "]").  Line 16 is a guard, if the condition is not
met, execution jumps somewhere else, in this case to the routine called
<Guard2> and is passed one parameter: p0.

Assuming we pass the guard, lines 17 through 23 are doing the dictionary lookup
to bracket_map to find where the program counter should jump to.  I'm not too
familiar with what the instructions are actually doing, but it looks like there
are two external calls and 3 guards. This seems quite expensive, especially
since we know bracket_map will never change (PyPy doesn't know that).  We'll
see below how to optimize this.

Line 24 increments the newly acquired instruction pointer. Lines 25 and 26 make
sure it's less than the program's length.

Additionally, line 27 guards that i21, the incremented instruction pointer, is
exactly 86. This is because it's about to jump to the beginning (line 29) and
the instruction pointer being 86 is a precondition to this block.

Finally, the loop closes up at line 28 so the JIT can jump to loop body <Loop0>
to handle that case (line 29), which is the beginning of the loop again. It
passes in parameters (p0, p1, i2, i3).

Optimizing
==========
As mentioned, every loop iteration does a dictionary lookup to find the
corresponding matching bracket for the final jump. This is terribly
inefficient, the jump target is not going to change from one loop to the next.
This information is constant and should be compiled in as such.

The problem is that the lookups are coming from a dictionary, and PyPy is
treating it as opaque. It doesn't know the dictionary isn't being modified or
isn't going to return something different on each query.

What we need to do is provide another hint to the translation to say that the
dictionary query is a pure function, that is, its output depends *only* on its
inputs and the same inputs should always return the same output.

To do this, we use a provided function decorator rpython.rlib.jit.purefunction,
and wrap the dictionary call in a decorated function::

    @purefunction
    def get_matching_bracket(bracket_map, pc):
        return bracket_map[pc]
        
This version can be found at `<example5.py>`_

Translate again with the JIT option and observe the speedup. Mandelbrot now
only takes 6 seconds!  (from 12 seconds before this optimization)

Let's take a look at the trace from the same function::

    [3c29fad7b792b0] {jit-log-opt-loop
    # Loop 0 : loop with 15 ops
    [p0, p1, i2, i3]
    debug_merge_point('+<[>[_>_+<-]>.[<+>-]<<-]++++++++++.', 0)
    debug_merge_point('+<[>[>_+_<-]>.[<+>-]<<-]++++++++++.', 0)
    i4 = getarrayitem_gc(p1, i2, descr=<SignedArrayDescr>)
    i6 = int_add(i4, 1)
    setarrayitem_gc(p1, i2, i6, descr=<SignedArrayDescr>)
    debug_merge_point('+<[>[>+_<_-]>.[<+>-]<<-]++++++++++.', 0)
    debug_merge_point('+<[>[>+<_-_]>.[<+>-]<<-]++++++++++.', 0)
    i7 = getarrayitem_gc(p1, i3, descr=<SignedArrayDescr>)
    i9 = int_sub(i7, 1)
    setarrayitem_gc(p1, i3, i9, descr=<SignedArrayDescr>)
    debug_merge_point('+<[>[>+<-_]_>.[<+>-]<<-]++++++++++.', 0)
    i10 = int_is_true(i9)
    guard_true(i10, descr=<Guard2>) [p0]
    debug_merge_point('+<[>[_>_+<-]>.[<+>-]<<-]++++++++++.', 0)
    jump(p0, p1, i2, i3, descr=<Loop0>)
    [3c29fad7ba32ec] jit-log-opt-loop}
    
Much better! Each loop iteration is an add, a subtract, two array loads, two
array stores, and a guard on the exit condition. That's it! This code doesn't
require *any* program counter manipulation.

I'm no expert on optimizations, this tip was suggested by Armin Rigo on the
pypy-dev list. Carl Friedrich has a series of posts on how to optimize your
interpreter that are also very useful: http://bit.ly/bundles/cfbolz/1

Final Words
===========
I hope this has shown some of you what PyPy is all about other than a faster
implementation of Python.

For those that would like to know more about how the process works, there are
several academic papers explaining the process in detail that I recommend. In
particular: Tracing the Meta-Level: PyPy's Tracing JIT Compiler.

See http://readthedocs.org/docs/pypy/en/latest/extradoc.html
