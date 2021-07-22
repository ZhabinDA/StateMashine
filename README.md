# StateMashine
using System;
using System.Net.Http;
using System.Runtime.CompilerServices;
using System.Threading.Tasks;
 
namespace TestAsyncAwaitStateMachine
{
    class Program
    {
        static async Task Main(string[] args)
        {
            await TestTask();
            await TestDummy();
 
            await TestDummyAwaitingForActivation();
        }
        //то что развернул с async/await в стейт машину
        public static async Task TestTask()
        {
            //sync code
            using var client = new HttpClient();
 
            //state 0
            string result1 = await client.GetStringAsync("https://www.youtube.com");
            //state 1
            string result2 = await client.GetStringAsync("https://www.instagram.com");
            //state 2
            Console.WriteLine($"{result1.Length} {result2.Length}");
 
        }
 
        //бесконечный авейт.
        //хотя все оп гайду https://youtu.be/_suxE9frTFA?t=1058
        private static Task TestDummyAwaitingForActivation()
        {
            var stateMachine = new StateMachine();
            stateMachine.Builder = AsyncTaskMethodBuilder.Create();
            stateMachine.Builder.Start(ref stateMachine); //внутри таска, которая проходит по всей стейт машине
            return stateMachine.Builder.Task; //тут уже другая таска которую никто не заресолвит.
        }
 
        //тут работает
        private static Task TestDummy()
        {
            var stateMachine = new StateMachine();
            stateMachine.Builder = AsyncTaskMethodBuilder.Create();
            var task = stateMachine.Builder.Task; //ибо таску сохранили заранее
            stateMachine.Builder.Start(ref stateMachine);
            return task;
        }
 
        
 
        private struct StateMachine : IAsyncStateMachine
        {
            public AsyncTaskMethodBuilder Builder;
            public int State;
            private TaskAwaiter<string> awaiter1;
            private TaskAwaiter<string> awaiter2;
 
            HttpClient client;
            public void MoveNext()
            {
                try
                {
                    switch (State)
                    {
                        case 0:
                            client = new HttpClient();
                            awaiter1 = client.GetStringAsync("https://www.youtube.com").GetAwaiter();
 
                            if (!awaiter1.IsCompleted)
                            {
                                this.State = 1;
                                var sm = this;
                                Builder.AwaitUnsafeOnCompleted(ref awaiter1, ref sm);
                                return;
                            }
 
                            goto case 1;
 
                        case 1:
                            awaiter2 = client.GetStringAsync("https://www.instagram.com").GetAwaiter();
                            
                            if (!awaiter2.IsCompleted)
                            {
                                this.State = 2;
                                var sm = this;
                                Builder.AwaitUnsafeOnCompleted(ref awaiter2, ref sm);
                                return;
                            }
 
                            goto case 2;
 
                        case 2:
                             
                            var s1 = awaiter1.GetResult();
                            var s2 = awaiter2.GetResult();
                            Console.WriteLine($"{s1.Length} {s2.Length}");
                            client.Dispose();
 
                             
                            State = -2;
                            this.Builder.SetResult();
                            break;
 
                    }
                }
                catch (Exception e)
                {
                    State = -2;
                    Builder.SetException(e);
                
                }
 
            }
 
            public void SetStateMachine(IAsyncStateMachine stateMachine)
            {
               
            }
        }
    }
}
