A Prefix
The prefix of the kernel uses three latitudes to define the number of threads in a thread group. If only 2 latitudes are used, the last parameter is 1.
[numthreads(thread_group_size_x,thread_group_size_y,1)]


GroupID: thread group id
ThreadIID: group thread id
DispatchThreadID: thread DispatchId
( DispatchThreadID = GroupID* group thread + ThreadId )

These ids are all started from 0. The two resource types
cs can be read. They take two types of resources: buffers, textures

[Buffer]
For example, the vertex buffer is a buffer. 
Many times we define the struct array and pass it as a buffer

To define the buffer itself ( which must be of a fixed size so not redefined later ) 
define:

struct Data
{
float x;
};

StructuredBuffer< Data > b;
RWStructuredBuffer< Data > b;

The addition of buffer is an append buffer , the consumption is a consume buffer


  Texture:
Read-only Texture2d< float4 > xx;
Read and write RWTexture2d< float4 > xx;
RWTexture2d< float2 > xx; //RG_int
  
 

Reading and writing in Unity can only be RenderTexture and support random read and write (RenderTexture enableRandomWrite=true)

THREE OTHER 
1 Each thread has a corresponding id called a : SV_DispatchThreadID
The Sampling Texture can't use Sample but SampleLevel. The extra parameter is mipmap level, 0 is the most advanced, 1 is secondary, 2...

2 int lattice converted to [0,1] uv range such as a 512x512 texture

Texture2d tex;
tex.SampleLevel(samPoint,float2(id.x,id.y)/512)


Blur requires all the pixels to be sampled, so it needs to be synchronized: GroupMemoryBarrierWithGroupSync();


[Example 1: Basic map calculation]
 

It's easy to assign all the pixels of a texture to red.

CS script
Using UnityEngine;
Using System.Collections;
Public class SetTexColor_1 : MonoBehaviour {
    Public material mat;
    Public ComputeShader shader;
    Void Start()
    {
        RunShader ();
    }
    Void RunShader()
    {
        /////////////////////////////////////////////
        // RenderTexture
        /////////////////////////////////////////////
        //1 New RenderTexture
        RenderTexture tex = new RenderTexture (256, 256, 24);
        //2 turn on random write
        tex.enableRandomWrite = true;
        //3 Create RenderTexture
        tex.Create ();
        //4 assigns a material
        mat.mainTexture = tex;
        /////////////////////////////////////////////
        // Compute Shader
        /////////////////////////////////////////////
        //1 find the KernelID to be used in the compute shader
        Int k = shader.FindKernel ("CSMain");
        //2 Set the texture parameter 1=kid parameter 2=the corresponding buffer name in the shader Parameter 3=corresponding texture, if you want to write the texture, the texture must be RenderTexture and enableRandomWrite
        shader.SetTexture (k, "Result", tex);
        //3 Run shader parameter 1=kid parameter 2=number of thread group in x dimension Parameter 3=number of thread group in y dimension Parameter 4=number of thread group in z dimension
        shader.Dispatch (k, 256 / 8, 256 / 8, 1);
    }
}


Compute Shader


//1 defines the name of the kernel
#pragma kernel CSMain
//2 define buffer
RWTexture2D Result;
//3 kernel function
/ / Three-dimensional thread number within the group
[numthreads(8,8,1)]
Void CSMain (uint3 id : SV_DispatchThreadID)
{
    / / Assign a value to the buffer
    //Pure red
   //Result[id.xy] = float4(1,0,0,1);
    / / Based on uv x give color
    Float v = id.x/256.0f;
    Result[id.xy] = float4(v,0,0,1);
}


This example is not about implementing a particle system, but just demonstrating the use and delivery of simple Buffers.

step:
Shader:
1 Define the struct structure
2 declare the struct variable
3 function calculation

c#:
1 Define the corresponding struct structure
2 declare struct array
3 create a buffer
   
   ComputeBuffer buffer = new ComputeBuffer(count,40);
   
 Parameter 1 is the length of the array (equal to the product of 2 dimensions), parameter 2 is the length of the structure, such as float=4

4 initialize the structure and give the buffer

buffer.SetData (values);

The argument is an struct array

5 Dispatch

Or FindKernel-> SetBuffer ->Dispatch


Compute Shader
#pragma kernel CSMain
struct PBuffer
{
    float life;
    float3 pos;
    float3 scale;
    float3 eulerAngle;
};
RWStructuredBuffer buffer;
float deltaTime;
[numthreads(2,2,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
     int index = id.x + id.y * 2 * 2;
     buffer[index].life -= deltaTime;
    buffer[index].pos = buffer[index].pos + float3(0,deltaTime,0); 
    buffer[index].scale = buffer[index].scale; 
    buffer[index].eulerAngle = buffer[index].eulerAngle + float3(0,20*deltaTime,0); 
}


CS脚本
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

//Buffer数据结构
struct PBuffer
{
    //size 40
    public float life;//4
    public Vector3 pos;//4x3
    public Vector3 scale;//4x3
    public Vector3 eulerAngle;//4x3
};

public class Particles_2 : MonoBehaviour {
    public ComputeShader shader;
    public GameObject prefab;
    private List<<font face="Consolas"> GameObject > pool = new List<<font face="Consolas"> GameObject >();
    int count = 16;
    private ComputeBuffer buffer;

    void Start()
    {
        for (int i = 0; i <<font face="Consolas">  count; i++) {
            GameObject obj = Instantiate (prefab) as GameObject;
            pool.Add (obj);
        }
        CreateBuffer ();
    }

    void CreateBuffer()
    {
        buffer = new ComputeBuffer(count,40);
        PBuffer[] values = new PBuffer[count];
        for (int i = 0; i <</span> count; i++) {
            PBuffer m = new PBuffer ();
            InitStruct (ref m);
            values [i] = m;
        }
        buffer.SetData (values);
    }

    void InitStruct(ref PBuffer m )
    {
        m.life = Random.Range(1f,3f);
        m.pos = Random.insideUnitSphere * 5f;
        m.scale = Vector3.one * Random.Range(0.3f,1f);
        m.eulerAngle = new Vector3 (0, Random.Range(0f,180f), 0);
    }
    void Update()
    {
        //运行Shader
        Dispatch ();

        //根据Shader返回的buffer数据更新物体信息
        PBuffer[] values = new PBuffer[count];
        buffer.GetData(values);
        bool reborn = false;
        for (int i = 0; i <</span> count; i++) {
            if (values [i].life <</span> 0) {
                InitStruct (ref values [i]);
                reborn = true;
            } else {
                pool [i].transform.position = values [i].pos;
                pool [i].transform.localScale = values [i].scale;
                pool [i].transform.eulerAngles = values [i].eulerAngle;
                //pool [i].GetComponent<</span>MeshRenderer>().material.SetColor ("_TintColor", new Color(1,1,1,values [i].life));
            }
        }
        if(reborn)
            buffer.SetData(values);
    }

    void Dispatch()
    {
        shader.SetFloat ("deltaTime", Time.deltaTime);
        int kid = shader.FindKernel ("CSMain");
        shader.SetBuffer (kid, "buffer", buffer);
        shader.Dispatch (kid, 2, 2, 1);
    }

    void ReleaseBuffer()
    {
        buffer.Release();
    }
    private void OnDisable()
    {
        ReleaseBuffer();
    }
}


    


