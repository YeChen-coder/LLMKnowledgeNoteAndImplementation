用 asyncio 启动一个异步的 main() 函数，在里面等待 Runner.run(...) 执行完成，然后打印最终结果。(async/await 的一个核心意义是：等待 I/O 的时候，可以把执行权让给其他异步任务。)
import asyncio
from agents import Agent, Runner

agent = Agent(
    name="History Tutor",
    instructions="You answer history questions clearly and concisely.",
)
async def main(): //async def 表示定义一个“异步函数”，也叫 coroutine function（协程函数）。用了async并不会像普通函数一样顺序的立即把里面所有代码执行完，而是得到一个 coroutine /ˌkəuru:'ti:n/对象。通常需要通过：**await main()** 或者：**asyncio.run(main())** 来真正运行它。async 通常就是 asynchronous /ei'siŋkrənəs/（异步）的缩写
    result = **await** Runner.run(agent, "When did the Roman Empire fall?")  //await 可以理解成：等这个异步操作完成，然后拿到它返回的结果。启动 Runner.run(...)-> 这个操作可能需要一段时间-> 等待结果-> 结果回来以后-> 赋值给 result; 假设 Runner.run() 是在调用 AI Agent，那么它可能需要：发送请求→ 等服务器响应→ Agent 处理→ 返回结果,网络请求可能需要几百毫秒甚至几秒。如果完全使用同步程序，那么“等待网络”的这段时间 Python 线程通常只能干等。这里面其实是异步套异步。
    
    print(result.final_output)

if __name__ == "__main__":
    **asyncio.run(main())**



