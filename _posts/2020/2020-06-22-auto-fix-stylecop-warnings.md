---
layout: single
classes: wide
title:  "Auto Fix StyleCop Warnings with Free Extensions"
date:   2020-06-08 09:00:00 -0400
permalink: blog/auto-fix-stylecop-warnings-with-free-extensions
header:
  teaser: /images/2020/auto-fix-stylecop-warnings/teaser-500x300.png
---

If you are like me, you like the idea of StyleCop. It helps ensure your code is consistently organized and your diffs are smaller. But the fact is, with standard Visual Studio you end up having to do many fixes by hand. Or, you have to run the auto fix for each warning type individually. This becomes a big problem if you also want to enforce StyleCop warnings (with gated code check-in or a step in your CI build) and want your team to stay happy. Thankfully, there are *free extensions* which can eliminate most of this pain. 

The two extensions I took a look at were:
* Code Maid - This is one of the most popular extensions in the Visual Studio Marketplace, and has a wide range of functionality.
* Code Formatter - Is an alternative which is more tightly focused on fixing StyleCop issues.

### Stress Test

The first big question with tools like this is what they can actually fix.
To that end I prepared a file with as many different StyleCop warnings as I could. Not all warnings were mutually compatible, but the resulting file should be enough for our purposes. That file is all the way at the bottom of the post. *I originally intended to to organize the warnings by type in the file and ended up giving that up. The file is just a disaster... but that was the point.*

If you are interested in playing with the file, the only special consideration is the `UnsafeMethod` method. If you don't normally write unsafe code, you can go ahead and remove that method.

For this test all StyleCop rules were left enabled (I normally disable a few). I also made a few changes to the settings of each tool to try to bring their behavior into alignment.

### Stress Test Results

This table contains one row for every StyleCop warning produced by the stress test file. The columns for each tool indicate whether they fixed some or all occurrences of an issue. I should note that most errors only occurred only once in the file. I included 'some' because a warning like SA1009 can usually be corrected automatically by both tools. However, in generating some of the other warnings I included unusual cases which these tools couldn't handle. 

This isn't a perfect test, so the best way to read these results is not as an exact description of the capabilities, but as a general representation of what they can accomplish. In this respect the tools are quite comparable. Indeed, what they can and can't fix makes quite a lot of sense.

* Both do a very good job handling whitespace.
* They can't add or change text, so they never fix things like variable or type names and they won't prefix local calls with `this.`.
* They generally don't move text between lines.
* They don't reorder or change your code within a method or statement. So changing `Nullable<int>` to `int?` or `readonly internal` to `internal readonly` will not happen. To be clear, they will make changes like reordering methods so public come before private.

|Warning|Detail|Code Formatter |Code Maid| 
|---| --- |:---:|:---:|
|SA0001|XML comment analysis is disabled due to project configuration|||
|SA1001|Commas should be followed by whitespace.|Some|Some|
|SA1002|Semicolons should not be preceded by a space.|Some|All|
|SA1003|Operator '+' should be preceded by whitespace.|All|All|
|SA1004|Documentation line should begin with a space.|||
|SA1005|Single line comment should begin with a space.|||
|SA1006|Preprocessor keyword 'if' should not be preceded by a space.|All||
|SA1007|Operator keyword should be followed by a space.|All|All|
|SA1008|Opening parenthesis should not be followed by a space.|All|All|
|SA1009|Closing parenthesis should not be followed by a space.|Some|Some|
|SA1010|Opening square brackets should not be preceded by a space.|All|All|
|SA1011|Closing square bracket should not be preceded by a space.|All|All|
|SA1012|Opening brace should be preceded by a space.|All|All|
|SA1013|Closing brace should be preceded by a space.|All|All|
|SA1014|Opening generic brackets should not be followed by a space.|All|All|
|SA1015|Closing generic bracket should be followed by a space.|All|All|
|SA1016|Opening attribute brackets should not be followed by a space.|All|All|
|SA1017|Closing attribute brackets should not be preceded by a space.|All|All|
|SA1018|Nullable type symbol should not be preceded by a space.|All|All|
|SA1019|Member access symbol '.' should not be preceded by a space.|All|All|
|SA1020|Increment symbol '++' should not be preceded by a space.|All|All|
|SA1021|Negative sign should not be followed by a space.|All|All|
|SA1022|Positive sign should not be followed by a space.|All|All|
|SA1023|Dereference symbol '*' should not be preceded by a space.|All|All|
|SA1024|Colon should be preceded by a space.|All|All|
|SA1025|Code should not contain multiple whitespace characters in a row.|All|All|
|SA1026|The keyword 'new' should not be followed by a space or a blank line.|All|All|
|SA1027|Tabs and spaces should be used correctly||All|
|SA1028|Code should not contain trailing whitespace|Some|All|
|SA1100|Do not prefix calls with base unless local implementation exists|||
|SA1101|Prefix local calls with this|||
|SA1102|Query clause should follow previous clause.|||
|SA1103|Query clauses should be on separate lines or all on one line|All|All|
|SA1104|Query clause should begin on new line when previous clause spans multiple lines|All|All|
|SA1105|Query clauses spanning multiple lines should begin on own line|All|All|
|SA1106|Code should not contain empty statements|||
|SA1107|Code should not contain multiple statements on one line|||
|SA1108|Block statements should not contain embedded comments|||
|SA1110|Opening parenthesis or bracket should be on declaration line.|||
|SA1111|Closing parenthesis should be on line of last parameter|||
|SA1112|Closing parenthesis should be on line of opening parenthesis|||
|SA1113|Comma should be on the same line as previous parameter.|||
|SA1114|Parameter list should follow declaration|||
|SA1115|The parameter should begin on the line after the previous parameter.|||
|SA1116|The parameters should begin on the line after the declaration, whenever the parameter span across multiple lines|||
|SA1117|The parameters should all be placed on the same line or each parameter should be placed on its own line.|||
|SA1118|The parameter spans multiple lines|||
|SA1119|Statement should not use unnecessary parenthesis|||
|SA1120|Comments should contain text||All|
|SA1121|Use built-in type alias|All||
|SA1122|Use string.Empty for empty strings|||
|SA1123|Region should not be located within a code element.|All||
|SA1124|Do not use regions|All||
|SA1125|Use shorthand for nullable types|||
|SA1127|Generic type constraints should be on their own line|||
|SA1128|Put constructor initializers on their own line|||
|SA1129|Do not use default value type constructor|||
|SA1130|Use lambda syntax|||
|SA1131|Constant values should appear on the right-hand side of comparisons|||
|SA1132|Each field should be declared on its own line|||
|SA1133|Each attribute should be placed in its own set of square brackets.|||
|SA1134|Each attribute should be placed on its own line of code.|All|All|
|SA1135|Using directive for namespace 'System.IO' should be qualified|All|All|
|SA1136|Enum values should be on separate lines|||
|SA1137|Elements should have the same indentation|||
|SA1139|Use literal suffix notation instead of casting|||
|SA1200|Using directive should appear within a namespace declaration|Some|Some|
|SA1201|A field should not follow a enum|||
|SA1202|'internal' members should come before 'private' members|||
|SA1204|Static members should appear before non-static members|Some||
|SA1205|Partial elements should declare an access modifier|All||
|SA1206|The 'internal' modifier should appear before 'readonly'|||
|SA1207|The keyword 'protected' should come before 'internal'.||All|
|SA1208|Using directive for 'System.Linq' should appear before directive for 'System.Runtime.CompilerServices'|All|All|
|SA1209|Using alias directives should be placed after all using namespace directives.|All|All|
|SA1210|Using directives should be ordered alphabetically by the namespaces.|All||
|SA1211|Using alias directive for 'C' should appear before using alias directive for 'T'|All|All|
|SA1212|A get accessor appears after a set accessor within a property or indexer.|||
|SA1214|Readonly fields should appear before non-readonly fields|||
|SA1216|Using static directives should be placed at the correct location.|All|All|
|SA1217|The using static directive for 'System.Math' should appear after the using static directive for 'System.Console'|All|All|
|SA1300|Element 'number1' should begin with an uppercase letter|||
|SA1302|Interface names should begin with I|||
|SA1303|Const field names should begin with upper-case letter.|||
|SA1306|Field 'Field1' should begin with lower-case letter|||
|SA1307|Field 'errors' should begin with upper-case letter|||
|SA1308|Field 's_Name' should not begin with the prefix 's_'|||
|SA1309|Field '_errors3' should not begin with an underscore|||
|SA1311|Static readonly fields should begin with upper-case letter|||
|SA1312|Variable 'Str' should begin with lower-case letter|||
|SA1314|Type parameter names should begin with T|||
|SA1400|Element 'foo' should declare an access modifier|All|All|
|SA1401|Field should be private|||
|SA1403|File may only contain a single namespace|||
|SA1405|Debug.Assert should provide message text|||
|SA1407|Arithmetic expressions should declare precedence|||
|SA1408|Conditional expressions should declare precedence|||
|SA1411|Attribute constructor should not use unnecessary parenthesis|||
|SA1413|Use trailing comma in multi-line initializers|||
|SA1500|Braces for multi-line statements should not share line|||
|SA1501|Statement should not be on a single line|||
|SA1502|Element should not be on a single line|Some||
|SA1505|An opening brace should not be followed by a blank line.|Some|All|
|SA1506|Element documentation headers should not be followed by blank line|All||
|SA1507|Code should not contain multiple blank lines in a row|Some|All|
|SA1508|A closing brace should not be preceded by a blank line.|Some|All|
|SA1509|Opening braces should not be preceded by blank line.|||
|SA1510|'catch' statement should not be preceded by a blank line||All|
|SA1512|Single-line comments should not be followed by blank line|||
|SA1514|Element documentation header should be preceded by blank line|All|All|
|SA1515|Single-line comment should be preceded by blank line|||
|SA1516|Elements should be separated by blank line|All|All|
|SA1517|Code should not contain blank lines at start of file|All|All|
|SA1519|Braces should not be omitted from multi-line child statement|||
|SA1520|Use braces consistently|||
|SA1600|Elements should be documented|||
|SA1601|Partial elements should be documented|||
|SA1602|Enumeration items should be documented|||
|SA1604|Element documentation should have summary|||
|SA1606|Element documentation should have summary text|||
|SA1607|Partial element documentation should have summary text|||
|SA1611|The documentation for parameter 'value' is missing|||
|SA1615|Element return value should be documented|||
|SA1626|Single-line comments should not use documentation style slashes|All||
|SA1629|Documentation text should end with a period|||
|SA1633|The file header is missing or not located at the top of the file.|||
|SA1649|File name should match first type name.|||

### Recommendation

In practice, either of these tools should handle the vast majority of StyleCop warnings that you generate on a day to day basis. So, I highly recommend using using one of these tools. Between the two, I would recommend Code Maid for a few reasons:

* Code Maid has a wider set of configurations options, so you should be able to have it closely meet your team's needs.
* Code Maid can export a file with those configurations. You can then share that file across your team so everyone's code clean-up is done the same way.
* Code Maid can clean up comments so that each line is a consistent length. I have found this to be a big time savings, and I am more likely to edit a comment I see that's inaccurate or incomplete, because I don't have to spend time fiddling with line lengths.
* Code Maid can be set to automatically do all of this when files are saved. There is clearly a hit to performance (which Visual Studio complains about on my machine), but I have enjoyed the extra bit of automation, compared to manually kicking off a cleanup. If performance is a concern, check out Code Formatter, in my testing it felt quicker.

### Stress Test File

``` csharp
 
using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using T = System.Text ;
using C = System.Runtime.CompilerServices;
using System.Linq;

using System.ComponentModel.DataAnnotations;
using Newtonsoft.Json;
using static System.Math;
using static System.Console;
using System.Threading;
using System.Threading.Tasks;
using System.Diagnostics;

namespace System.Threading
{
    using IO;
    using Tasks;
}

namespace StyleCopAutoFixStressTest
{


    public class StressTest:IDisposable
    {
        public enum TestEnum { A,B}
        private int Field1,
    field2; // SA1132
        internal int errors;
        readonly internal int _errors3;
        internal protected int errors2;
        protected int count;
        const string foo = "foo";
        int nNumber;

        private string s_Name = "sdf";
        static readonly string name = "sdf";

        public readonly int errors5 = 5;
        public interface badInterface { }


        [ Required(), MaxLength(5) ][MinLength(2)]
        public int ?  NullableInt { get; set; }

        public void Dispose()
        {
            throw new NotImplementedException();
        }

        public int number1 { set; get; }
        internal int Number2 { get; set; }
        private int Number3 { get; set; }

        partial class SubClass<Param>{}
        static public void  StaticVoid() => throw new NotImplementedException();
        /// <summary>
        ///Docs
        /// </summary>
        public void Spacing()
        {
            //single line comment should begin with space.

            var a = ( 1+2 )+2;
            a ++;
            var b = - 5;
            var c = + 5;
            var value = false;
            var list = new List< string > { "sdf" , "jsf" };
            list . Add("string");

            if (!value)
            {
            }

            var array = new int [ 0 ] ;

            if (true){var z = true;}

            // Tab is here -> 	 

            var d = new [] { 1, 10, 100, 1000 };

            base.GetType();
            var e = count;
        }

        static unsafe public void UnsafeMethod(int p, out int id)
        {
            int* i = (int * )(p);
            id = *i++;
        }

        public void Readability()
        {
            IList<string> stringList = new List<string>() {
    "C# Tutorials",
    "VB.NET Tutorials",
    "Learn C++"
    , "MVC Tutorials" ,
    "Java"
};

            var result = from g in stringList

                         where g.Contains("Tutorials")

                         select g;

            var result2 = from h in stringList where h.Contains("Tutorials")
select h;

            var elementNames =
    from element in GetElements(
        12,
                    45
    ) select TransformElement
    (
        element); ;


            if (true)
            #region
            // Make sure x does not equal y
            {
            }
            #endregion

                #region
            string GetName(
                )
            {
                return "";
            }
                #endregion

            void JoinName(

    string first, 
    
    string last)
            {
            }

            JoinName(
                "A",
                "B" +
                "c");

            //
            String s = "s";

            void ReJoinName(string first, string middle,
string last)
            {
                
            }

            IEnumerable<string> GetElements(int a
                , int b) => throw new NotImplementedException();
            string TransformElement(string s) => throw new NotImplementedException();

            Nullable<int> nullable = 5;
            DoSomething(); //Note, SA1126 is disabled by default

            void Method<T, R>() where T : class where R : class, new()
            {
            }

            ImmutableArray<int> array = new ImmutableArray<int>();
            Action a = delegate { var x = 0; };
            var x = (long)1;
            ValueTuple<int, int> t1; // SA1141
            (int valueA, int valueB) t2;

            var y = t2.Item1; // SA1142
        }

        private void Naming()
        {
            var s_str = "b";
            var Str = "b";
            (int ValueA, int ValueB) T2;
        }

        public (int ValueA, int ValueB) ExampleMethod() => throw new NotImplementedException();
        /// <summary>
        /// 
        /// </summary>
        internal void MaintainabilityRules()
        {
            var a = (1);
            int x = 5 + 4 * 3 / 6 % 4 - 2;

            if (true || false && false && true || true)
            {
            }

            Debug.Assert(!true);
            Debug.Fail(string.Empty) ;

            try
            {
            }
            catch (Exception ex)
            {
            }

            
        }

        /// <summary>
        /// Documentation for the first part of Class1.
        /// </summary>
        public partial class Class1
        {
        }

        /// <summary>
        /// </summary>
        public partial class Class1
        {
        }

        /// <param name="err">fsdf</param>

        public static bool Layout(bool err)
        {
            if (err) throw new Exception();
            else
            {
                var a = 1;
            }

            if (err)

            {

                throw new Exception();
            }

            try
            {
                
            }

            catch (Exception ex)
            {
                Console.WriteLine(ex.ToString());
            }

            while (err)

            {

            }

            if (true)
                return
                    err;
        }

        /// <return>
        /// returns void;
        /// </return>
        public void Method(string value)
        {
            if (null == value) // SA1131
            {
                throw new ArgumentNullException(nameof(value));
            }

            /// tripple slash comment
        }

        public class TypeName
        {
            public TypeName() : this(0)
            {
            }

            public TypeName(int value)
            {
            }
        }

        #region 
        private void DoSomething()
        {
        }
        #endregion

        public static StressTest operator +(StressTest a, StressTest b) => throw new NotImplementedException();

    }

# if Debug
#endif
}
```