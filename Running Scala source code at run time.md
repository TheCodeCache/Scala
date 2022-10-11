# Taking scala code as string and run dynamically at run time - 

```scala
package com.myorg.datalake.ingestion.etl

import java.io.File

import scala.collection.mutable.ListBuffer
import scala.io.Source

import org.apache.commons.io.FileUtils
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.StringType
import org.apache.spark.sql.types.StructField
import org.apache.spark.sql.types.StructType
import org.json4s.jackson.JsonMethods
import org.slf4j.Logger
import org.slf4j.LoggerFactory

import com.myorg.datalake.ingestion.common.Constants
import com.myorg.datalake.ingestion.common.Constants.CSV
import com.myorg.datalake.ingestion.common.Constants.DELIMITED
import com.myorg.datalake.ingestion.common.Constants.DataFile_Format
import com.myorg.datalake.ingestion.common.Constants.FIXED_WIDTH
import com.myorg.datalake.ingestion.common.Constants.JSON
import com.myorg.datalake.ingestion.common.Constants.TSV
import com.myorg.datalake.ingestion.common.Enums
import com.myorg.datalake.ingestion.exception.IngestionException
import com.myorg.datalake.ingestion.parser.DelimitedParser
import com.myorg.datalake.ingestion.parser.FixedWidthParser
import com.myorg.datalake.ingestion.parser.JsonParser
import com.myorg.datalake.ingestion.parser.JsonTextParser
import com.myorg.datalake.ingestion.util.CatalogReader
import com.myorg.datalake.ingestion.util.CatalogReader.FeedDataFileColumn
import com.myorg.datalake.ingestion.util.CatalogReader.FeedMetaFileColumn
import com.myorg.datalake.ingestion.util.SparkUtil
import com.myorg.datalake.ingestion.util.StringUtils
import com.myorg.datalake.voltage_integration.EncryptionUtil
import com.voltage.securedata.enterprise.VeException
import scala.collection.immutable.ListMap
import java.util.concurrent.atomic.AtomicInteger

object Dummy extends App {

  val logger: Logger = LoggerFactory.getLogger(Dummy.getClass)

  type DataFrame = org.apache.spark.sql.DataFrame

  //def function(x: Int): Int = { val y = 2 * x + 1; println(y); y * y }
  def function(x: String): String = { if (x.equals("ABC")) x + "abc" else x + "pqr" }
  //def function(x: Tuple2[String, String]): String = { x._2 + "-abc-" + x._1 }
  def function(x: Seq[Tuple2[String, String]]): String = { x(0)._2 + "-abc-" + x(0)._1 }

  //val func = """import org.apache.commons.io.FileUtils; import java.io.File; def function(x: Tuple4[Any, String, Int, Int]): Any = {println(FileUtils.sizeOfDirectory(new File("hadoop_log_dir"))); val (fieldValue, maskChar, start, end) = (x._1, x._2, x._3, x._4); if (fieldValue == null) "" else {val value = fieldValue.toString; val len = value.length; if (end >= len) value.patch(0, maskChar * (start), start) else value.patch(0, maskChar * (start), start).patch(end, maskChar * (len - end), len)} }"""

  //val func = """def function(x: Tuple2[Any, String]): Any = {EncryptionUtil.getInstance().encrypt(format, input)}"""

  //val func = """def function(x: Tuple2[String, String]): Any = { import com.myorg.datalake.voltage_integration.EncryptionUtil; val (format, input) = (x._1, x._2); EncryptionUtil.getInstance().encrypt(format, input) }"""

  val func1 = """def function(values: Seq[Any]): Any = {println("printing inside udf", values.size); values.foreach(println); "manoranjan"}"""

  def function(x: Tuple4[Any, String, Int, Int]): String = {
    val (fieldValue, maskChar, start, end) = (x._1, x._2, x._3, x._4);
    if (fieldValue == null) "" else {
      val value = fieldValue.toString;
      val len = value.length;
      if (end >= len) value.patch(0, maskChar * (start), start) else value.patch(0, maskChar * (start), start).patch(end, maskChar * (len - end), len)
    }
  }
  //"blank", "123", "true", "0.02f", "0.04"

  //////////////
  ////////////// MAIN ENTRY POINT /////////////
  test(List("blank", 123, true, 0.02f, 0.04))

  def test(aa: Seq[Any]) {

    val (a: String) :: (b: Int) :: (c: Boolean) :: (d: Float) :: (e: Double) :: (f: Seq[Any]) = aa
    //    val a = aa(0).toString
    //    val b = aa(1).toInt
    //    val c = aa(2).toBoolean
    //    val d = aa(3).toFloat
    //    val e = aa(4).toDouble

    println(a.getClass, a)
    println(b.getClass, b)
    println(c.getClass, c)
    println(d.getClass, d)
    println(e.getClass, e)

    def dosomestuff {
      import scala.reflect.runtime.universe._
      import scala.tools.reflect.ToolBox

      val mirror = runtimeMirror(this.getClass.getClassLoader)
      val tb = mirror.mkToolBox()

      val function = """def function(values: Seq[Any]): Any = {
  println("printing inside actual udf", values.size); 
  val (a:String)::(b:Int)::(c:Boolean)::(d:Double)::(e:List[Any]) = values;
  println(a,b,c,d,e);
  values.foreach(println); 
  "returns_some_value"
  }"""

      val functionWrapper = s"""object FunctionWrapper {${function}}"""
      val functionSymbol = tb.define(tb.parse(functionWrapper).asInstanceOf[tb.u.ImplDef])

      val args: List[Any] = List("abc", 12, true, 0.02) // list of values with different datatype
      def liftYolo: Liftable[Any] = new Liftable[Any] {
        def apply(value: Any): reflect.runtime.universe.Tree = {
          value match {
            case s: String  => Liftable.liftString(s)
            case i: Int     => Liftable.liftInt(i)
            case b: Boolean => Liftable.liftBoolean(b)
            case d: Double  => Liftable.liftDouble(d)
          }
        }
      }

      implicit def liftAnyList: Liftable[List[Any]] = Liftable.liftList(liftYolo)
      tb.eval(q"$functionSymbol.function($args)")
      //okay, but
      val badArgs: List[Any] = List(Map(1 -> "one"))
      tb.eval(q"$functionSymbol.function($badArgs)")
    }

    //dosomestuff
    //return

    val df = getDF()
    df.printSchema()
    df.show()

    def getLogic(): String = {
      //"""def function(fieldValue: String, maskChar: String, start: Int = 0, end: Int = 1000): String = { try {   if (fieldValue == null)  ""   else {  val value = fieldValue.toString;  val len = value.length;  if (end >= len)    value.patch(0, maskChar * (start), start)  else    value.patch(0, maskChar * (start), start).patch(end, maskChar * (len - end), len)   } } catch {   case ex: Exception => null }  }"""
      """def function(values: Seq[Any]): String = {
        println("actual values@@@@@@@@@",values);
      println(values(0).getClass);println(values(1).getClass);println(values(2).getClass);println(values(3).getClass);
      println("type: ",values.getClass);
      val (fieldValue:String)::(maskChar:String)::(start:Int)::(end:Int)::(e:List[Any]) = values; 
        try {   if (fieldValue == null)  ""   else {  val value = fieldValue.toString;  val len = value.length;  if (end >= len)    value.patch(0, maskChar * (start), start)  else    value.patch(0, maskChar * (start), start).patch(end, maskChar * (len - end), len)   } } catch {   case ex: Exception => null }  }"""
    }

    def parse(signature: String): ListMap[String, String] = {
      //ListMap(1 -> 2) + (3 -> 4)
      ListMap("AL" -> "Alabama")
    }

    def getSignature(argument: ListMap[String, String]): String = {
      ""
    }

    def manipulate(logic: String): String = {
      val bodyIdx = logic.indexOf('{')
      val funcLparenIdx = logic.indexOf('(')
      val funcRparenIdx = logic.indexOf(')')
      val arguments = logic.substring(funcLparenIdx + 1, funcRparenIdx - 1)
      println(arguments)
      val args: ListMap[String, String] = parse(arguments)
      val replacement = getSignature(args)
      logic.replaceFirst(arguments, replacement)
      ""
    }

    // ############### manipulated function #############
    val fun1 = getLogic()

    val fun2 = """def function(values: Seq[Any]): String = { 
      println("actual values#########",values);
      println(values(0).getClass);println(values(1).getClass);println(values(2).getClass);println(values(3).getClass);
      println("type: ",values.getClass);
      val (fieldValue:String)::(maskChar:String)::(start:Int)::(end:Int)::(e:List[Any]) = values; 
      try {   if (fieldValue == null)  ""   else {  val value = fieldValue.toString;  val len = value.length;  if (end >= len)    value.patch(0, maskChar * (start), start)  else    value.patch(0, maskChar * (start), start).patch(end, maskChar * (len - end), len)   } } catch {   case ex: Exception => null }  }"""

    val fun4 = """def function(values: Seq[Any]): String = {
      println("inside actual udf fun4..");
      import com.myorg.datalake.voltage_integration.EncryptionUtil; 
      val (format:String)::(input:String)::(e:List[Any]) = values; 
      EncryptionUtil.getInstance().encrypt(format, input) }"""

    val fun5 = """def function(values: Seq[Any]): Int = {
      println("inside actual udf fun5..");
      val (value:Int)::(e:List[Any]) = values; 
      value + 100000 }"""

    val fun6 = """def function(values: Seq[Any]): String = {
      println("inside actual udf fun6..");
      val (value:String)::(e:List[Any]) = values; 
      value }"""

    val fun7 = """def function(values: Seq[Any]): java.sql.Date = {
      val (value:String)::(e:List[Any]) = values;
    println("inside actual udf fun7..");
    import java.text.SimpleDateFormat
    import java.time.LocalDate
    import java.time.format.DateTimeFormatter
      val localDate = LocalDate.parse(value, DateTimeFormatter.ofPattern("yyyyMMdd")).format(DateTimeFormatter.BASIC_ISO_DATE) 
      val sqldateFormat = new java.text.SimpleDateFormat("yyyyMMdd"); 
      //val date = formatter.parse(value);
      val sqlDate = new java.sql.Date(sqldateFormat.parse(localDate).getTime)
      sqlDate }"""

    val fun8 = """
      import java.text.SimpleDateFormat
      import java.time.LocalDateTime
      import java.time.format.DateTimeFormatter
      def function(values: Seq[Any]): java.sql.Timestamp = {
      val (value:String)::(e:List[Any]) = values;
      println("inside actual udf fun8..");
      
      println("timestamp value:", value)
      val formatter = DateTimeFormatter.ofPattern("MM/dd/yyyy HH:mm:ss");
		  val localDateTime = LocalDateTime.from(formatter.parse(value));
		  java.sql.Timestamp.valueOf(localDateTime)
    }"""

    val fun9 = """def function(values: Seq[Any]): String = {
      val (value:String)::(e:List[Any]) = values;
      
		  value+"_abcdef"
    }"""

    // ############### LOGIC #############

    //return

    import org.apache.spark.sql.Column
    import org.apache.spark.sql.types._
    import org.apache.spark.sql.functions._
    import org.apache.spark.sql.functions.lit

    val spark = SparkUtil.getSparkSession("pag", "*")
    import spark.implicits._

    import scala.reflect.runtime.universe.{ Quasiquote, runtimeMirror, Liftable, showCode, showRaw }
    import scala.tools.reflect.ToolBox

    val mirror = runtimeMirror(this.getClass.getClassLoader)
    val tb = ToolBox(mirror).mkToolBox()

    //val function = """def function(values: Seq[String]): Any = { println("printing inside udf", values.size); values.foreach(println); "returns_some_value" }"""
    //val function = fun

    //    val idx = "1"
    //    val delegatef = s"""def function(values: Seq[String]) {function${idx}(values)}"""
    //
    //    val functionWrapper = s"""object FunctionWrapper {${delegatef}; ${fun1}; ${fun2}}"""
    //    println(functionWrapper)
    //    val functionSymbol = tb.define(tb.parse(functionWrapper).asInstanceOf[tb.u.ImplDef])

    //    val args = List("blank", "123", "true", "0.02f", "0.04")
    //    tb.eval(q"$functionSymbol.function($args)")
    //    val delegate = udf((args: Seq[String]) =>
    //      {
    //        tb.eval(q"$functionSymbol.function1(${args.toList})")
    //      }, StringType)

    def liftYolo: Liftable[Any] = new Liftable[Any] {
      def apply(value: Any): reflect.runtime.universe.Tree = {
        println("incoming1:", value, value.getClass)
        value match {
          case i: Int     => Liftable.liftInt(i);
          case b: Boolean => Liftable.liftBoolean(b);
          case d: Double  => Liftable.liftDouble(d);
          case s: String  => Liftable.liftString(s);
        }
      }
    }
    implicit def liftAnyList: Liftable[List[Any]] = Liftable.liftList(liftYolo)

    case class Argument(
      name:        String,
      dataType:    String,
      isColumn:    Boolean,
      position:    Int,
      initialized: Boolean,
      initialData: Option[Any])

    class Signature {
      private var logic: String = ""
      private var returnDataType: String = ""
      private var counter = new AtomicInteger(1)
      private var derivedFieldName = ""
      private var args: List[Argument] = Nil
      def addArg(arg: Argument): Signature = {
        args = arg :: args
        this
      }
      def setFieldName(name: String): Signature = {
        derivedFieldName = name;
        this
      }
      def getFieldName(): String = derivedFieldName

      def getArgs(): List[Argument] = args
      def getIdx(): Int = counter.getAndIncrement
      def addLogic(logic: String): Signature = { this.logic = logic; this }
      def code(): String = logic
      def addReturnType(rtype: String): Signature = { this.returnDataType = rtype; this }
      def returnType(): String = returnDataType
    }

    def getSigns(): List[Signature] = {
      var sigs: List[Signature] = Nil

      ???
    }

    def getSigs(): List[Signature] = {
      var sigs: List[Signature] = Nil
      val firstWord = Argument("first_word", "String", "true".toBoolean, "0".toInt, false, None)
      val maskChar = Argument("maskChar", "String", false, 1, true, Some("X"))
      val startIdx = Argument("startIdx", "Int", false, "2".toInt, true, Some("2".toInt))
      val endIdx = Argument("endIdx", "Int", false, 3, true, Some(6))
      val sig = new Signature().setFieldName("first_word_derived")
        .addArg(firstWord).addArg(maskChar).addArg(startIdx).addArg(endIdx).addLogic(fun1)
      sigs = sig :: sigs
      val secondWord = Argument("second_word", "String", true, 1, false, None)
      val format = Argument("format", "String", false, 0, true, Some("VLS-FPE-AlphaNum"))
      val sig2 = new Signature().setFieldName("second_word_derived").addArg(secondWord).addArg(format).addLogic(fun4)
      sigs = sig2 :: sigs
      val intValues = Argument("int_values", "Int", true, 0, false, None)
      val sig3 = new Signature().addReturnType("Int").setFieldName("int_values_derived").addArg(intValues).addLogic(fun5)
      sigs = sig3 :: sigs
      val staticValue = Argument("XYZ", "String", false, 0, true, Some("XYZ"))
      val sig4 = new Signature().addReturnType("String").setFieldName("static_values_derived").addArg(staticValue).addLogic(fun6)
      sigs = sig4 :: sigs
      val dateValue = Argument("date_field", "String", true, 0, false, None)
      val sig5 = new Signature().addReturnType("java.sql.Date").setFieldName("date_derived").addArg(dateValue).addLogic(fun7)
      sigs = sig5 :: sigs
      val timeValue = Argument("timestamp_field", "String", true, 0, false, None)
      val sig6 = new Signature().addReturnType("java.sql.Timestamp").setFieldName("timestamp_derived").addArg(timeValue).addLogic(fun8)
      sigs = sig6 :: sigs
      val endsWith = Argument("first_word", "String", true, 0, false, None)
      val sig7 = new Signature().setFieldName("endsWith_derived").addArg(endsWith).addLogic(fun9)
      sigs = sig7 :: sigs
      sigs
    }

    var signatures: List[Signature] = getSigs()

    val transformedDF = signatures.foldLeft(df) { (df, sign) =>
      {
        val colsBuff = ListBuffer[Column]()
        sign.getArgs().sortBy(_.position).foreach(arg => {
          //according to the position of arguments in method signature, this "cols" must be populated
          if (arg.isColumn)
            colsBuff += col(arg.name)
          else
            colsBuff += lit(arg.initialData.get)
        })
        val cols = colsBuff.toList
        println("!!!!!!!!!!!!!!! cols !!!!!!!!!!", cols)

        df.withColumn(sign.getFieldName(), {
          // one-time activity per function
          val functionWrapper = s"""object FunctionWrapper_${sign.getIdx()} {${sign.code()}}"""
          println(functionWrapper)
          val code = tb.parse(functionWrapper)
          //println(showRaw(code))
          println(showCode(code))
          val functionSymbol = tb.define(code.asInstanceOf[tb.u.ImplDef])
          println("helloworld-1")

          udf(
            (params: Row) =>
              tb.eval(q"$functionSymbol.function(${params.toSeq.toList})"),
            sign.returnType().toLowerCase() match {
              case "string"             => StringType
              case "int"                => IntegerType
              case "java.sql.date"      => DateType
              case "java.sql.timestamp" => TimestampType
              case _                    => StringType
            })(struct(cols: _*))
        })
      }
    }

    transformedDF.printSchema()
    transformedDF.show()
    println(":: transformed DF ::")
    return

    val columns = List("cola", "colb")

    val df2 = df.withColumn("first_word_derived", {
      val idx = "1"
      val args = List("first_word", "X", 2, 6)
      val cols = List(col(args(0).asInstanceOf[String]), lit(args(1)), lit(args(2)), lit(args(3)))

      val functionWrapper = s"""object FunctionWrapper${idx} {${fun1}}"""
      println(functionWrapper)
      val code = tb.parse(functionWrapper)
      println(showRaw(code))
      println(showCode(code))
      val functionSymbol = tb.define(code.asInstanceOf[tb.u.ImplDef])
      println("helloworld-1")

      udf((params: Row) =>
        {
          val args = params.toSeq

          tb.eval(q"$functionSymbol.function(${args.toList})")
        }, StringType)(struct(cols: _*))
    }).withColumn("second_word_derived", {
      val idx = "2"
      val args = List("second_word", "VLS-FPE-AlphaNum")
      val cols = List(lit(args(1)), col(args(0).asInstanceOf[String]))

      val functionWrapper = s"""object FunctionWrapper${idx} {${fun4}}"""
      println(functionWrapper)
      val code = tb.parse(functionWrapper)
      println(showRaw(code))
      println(showCode(code))
      val functionSymbol = tb.define(code.asInstanceOf[tb.u.ImplDef])
      println("helloworld-2")

      udf((params: Row) =>
        {
          val args = params.toSeq

          tb.eval(q"$functionSymbol.function(${args.toList})")
        }, StringType)(struct(cols: _*))
    })
    df2.printSchema()
    df2.show()

    return ;

    //    import scala.reflect.runtime.universe.{ Quasiquote, runtimeMirror }
    //    import scala.tools.reflect.ToolBox
    //
    //    val mirror = runtimeMirror(this.getClass.getClassLoader)
    //    val tb = ToolBox(mirror).mkToolBox()

    //    val data = Array("A", "B", "C")
    //
    //    println("Data before function applied on it")
    //    println(data.mkString(","))

    println("size of dir:", FileUtils.sizeOfDirectory(new File("hadoop_log_dir")))

    println("Please enter the map function you want:")
    //val function = scala.io.StdIn.readLine()
    //val function = func
    //    val functionWrapper = s"""object FunctionWrapper {${function}}"""
    //    println
    //    println
    //    println(functionWrapper)
    //    println
    //    println
    //    val functionSymbol = tb.define(tb.parse(functionWrapper).asInstanceOf[tb.u.ImplDef])

    //val x = List(Tuple2("A", "B"), Tuple2("C", "D"))
    //val list = List(Tuple4("manoranjan", "X", 1, 5), Tuple4("kumar", "Z", 1, 5), Tuple4("helloworld", "Y", 1, 5))
    //val x = "ABC"
    //list.foreach(x => { val outs = tb.eval(q"$functionSymbol.function($x)"); println(outs) })

    //    val x = Tuple2("VLS-FPE-AlphaNum", "manoranjan")
    //    val outs = tb.eval(q"$functionSymbol.function($x)")
    //    println("encrypted value: ", outs)
    //
    //    val y = Tuple2("VLS-FPE-AlphaNum", "kumar")
    //    val outs2 = tb.eval(q"$functionSymbol.function($y)")
    //    println("encrypted value: ", outs2)

    // Map each element using user specified function
    //    val dataAfterFunctionApplied = data.map(x => tb.eval(q"$functionSymbol.function($x)"))
    //
    //    println("Data after function applied on it")
    //    println(dataAfterFunctionApplied.mkString(","))
    //val outs2 = tb.eval(q"$functionSymbol.function _").asInstanceOf[String => String]
    //println(outs)
    println
    println
    //    class DummyClass
    //    println
    //    import javax.script.ScriptEngineManager
    //    val engine = new ScriptEngineManager().getEngineByName("scala")
    //    val settings = engine.asInstanceOf[scala.tools.nsc.interpreter.IMain].settings
    //    settings.embeddedDefaults[DummyClass]
    //    engine.eval("val x: Int = 5")
    //    val thing = engine.eval("x + 9").asInstanceOf[Int]
  }

  def getDF(): DataFrame = {
    val spark = SparkUtil.getSparkSession("pag", "*")
    import spark.implicits._
    val df = Seq(
      ("developer", "architect", 123, 1824.5, true, "20180213", "06/22/2011 11:29:41"),
      ("hihellohey", "helloworld", 12, 124.55, false, "20160423", "02/12/2014 21:19:41"),
      ("myorg-project", "transamerica", 1223, 1243.5, false, "20171212", "06/22/2011 11:29:42"),
      ("transamerica", "released", 1623, 2124.5, true, "20151231", "12/30/2021 01:35:52"))
      .toDF("first_word", "second_word", "int_values", "double_values", "flag", "date_field", "timestamp_field")
    df
  }

  println
  //scala.util.Try(ingestIt())
  println("\n\n\n==============================================\n\n\n")
  //readORCFile("C:\\Z\\unified_layer_master_data\\")
  //poc

  //  val spark = SparkUtil.getSparkSession("test")
  //  val file = "C:\\Z\\part-00001-0da8cd07-7f75-4911-b7de-1ce53158494f.c000.snappy.orc"
  //  //val file = "C:\\Z\\part-00005-c0963276-f270-42f6-ace8-870981e3fa0f.c000.snappy.orc"
  //
  //  val csvDF = spark.read.format("orc").load(file)
  //
  //  csvDF.printSchema
  //  csvDF.show(1000, false)

  def ingestIt() {
    val spark = SparkUtil.getSparkSession("pag", "*")
    //    def input(i: Int) = spark.sparkContext.parallelize(1 to i * 100000)
    //    def serial = (1 to 10).map(i => input(i).reduce(_ + _)).reduce(_ + _)
    //    def parallel = (1 to 10).map(i => Future(input(i).reduce(_ + _))).map(Await.result(_, 10.minutes)).reduce(_ + _)
    try {
      FileUtils.deleteDirectory(new File("hadoop_log_dir"))

      IngestionContext.sc = spark.sparkContext
      IngestionContext.sparkSession = spark

      val map = scala.collection.mutable.Map[String, String]()

      val paramMap = scala.collection.mutable.Map[String, String]()

      paramMap(Constants.SOR) = "default"
      paramMap(Constants.DATA_INGESTION) = "Data Ingestion"
      paramMap(Constants.INGESTION_ID) = "2391"
      paramMap(Constants.HADOOP_LOG_DIR) = "hadoop_log_dir"
      paramMap(Constants.Response_Path) = "hadoop_log_dir/response"
      paramMap(Constants.Encryption_Key) = "alias/S3/myorgDev-MBDL"
      paramMap(Constants.REJECTED_RECORDS_BUCKET) = "artifacts"
      paramMap(Constants.REJECTED_RECORDS_DB) = "reference"
      paramMap(Constants.REJECTED_RECORDS_TABLE) = "global_rejected_rcords"

      try {
        val encSaltKey = EncryptionUtil.getInstance().encrypt("VLS-FPE-AlphaNum", "myorgDev")

        paramMap("salt_key") = encSaltKey

        FileUtils.deleteDirectory(new File("hadoop_log_dir/bad_records"))
      } catch {
        case ex: VeException =>
          println("VEEXCEPTION:", ex)
          ex.printStackTrace()
        case ex: Exception =>
          println("EXCEPTION:", ex)
          ex.printStackTrace()
      }

      //val arr = Array("ctl_e022252d_053120_20200605_222.0_1.0", "ctl_e022252d_053120_20200605.data", "1", "fixed_width")
      //val arr = Array("transamericausplbrokerpl_search_customer_response_20200706084144_541.0_1.0", "transamericausplbrokerpl_search_customer_response_20200706084144.data", "100")
      //val arr = Array("transamericausplbrokerpl_policy_request_20200716082900_544.0_1.0", "transamericausplbrokerpl_policy_request_20200716082900.data", "100")
      //val arr = Array("transamericausplbrokerpl_coverages_response_20200612113835_547.0_1.0", "transamericausplbrokerpl_coverages_response_20200612113835.data", "100")
      //val arr = Array("audit_trail_20200630165623_10.0_1.0", "audit_trail_20200630165623.json", "100", "json")
      //val arr = Array("Data_Element_Standardization_Mapping_257.0_1.0", "Data_Element_Standardization_Mapping.json", "100", "json")
      //val arr = Array("m2brokernlvictornl_victor_quote_response_20200722150249_375.0_1.0", "m2brokernlvictornl_victor_quote_response_20200723110221.data", "100")
      //val arr = Array("icd_e029113d_123119_20200108_241.0_1.0", "icd_e026798d.063020_20200506.pgp.decrypt.data", "100", "fixed_width")
      //val arr = Array("claimpretriages_20200827_153946_189.0_1.0", "xaa.data", "100", "json")

      val arr = Array("LOSS_DETAIL_20190917_1.0_4.0", "LOSS_DETAIL_20190917.data", "100")
      //val arr = Array("Travelers6_255.0_1.0", "Travelers6.json", "100", "json")
      //val arr = Array("Itron_255.0_1.0", "Itron.json", "100", "json")
      //      val arr = Array("accgrp_121.0_1.0", "accgrp.json", "100", "json")
      //val arr = Array("m2brokernlvictornl_victor_quote_request_20200410090828_374.0_1.0", "m2brokernlvictornl_victor_quote_request_20200410090828.data", "10")

      map("catalog") = arr(0)
      map("file") = arr(1)
      map("count") = arr(2)
      map("format") = if (arr.length == 4) arr(3) else "delimited"
      IngestionContext.paramMap = paramMap.toMap
      val t = parseCatalog(map("catalog"))

      val (pm, dataFileColList, metaFileColList) = t

      val dataFilePath = "/abc/def/" + map("file")

      val pair = StringUtils.feedKeyNamePair(dataFilePath)
      logger.info(s"Feed Key:	${pair._2} and Feed File:	${pair._1}")
      logger.info("Feed Key: {} and Feed File:	{}", pair._2, pair._1, "")

      paramMap(Constants.FEED_KEY) = pair._2.trim
      paramMap(Constants.FEED_FILE) = pair._1.trim
      paramMap(Constants.SALT_KEY) = "EZR0FFVL"
      paramMap(Constants.DataFile_Format) = map("format")
      paramMap(Constants.MetaFile_Path) = "C:\\Z\\fixed_width1.meta"

      paramMap ++= pm

      logger.info("all paramMap contents:")
      paramMap.foreach(pair => logger.info("	{} : {}", pair._1, pair._2, ""))

      IngestionContext.sc = spark.sparkContext
      IngestionContext.dataFileColList = dataFileColList
      IngestionContext.metaFileColList = metaFileColList
      IngestionContext.paramMap = paramMap.toMap
      IngestionContext.sparkSession = spark
      IngestionContext.expectedRecordsCount = map("count").toInt

      //      val (dataFile: String, expectedRecordCount: Long) = MetaFileReader.readMataFile(spark.sparkContext, metaFileColList, paramMap.toMap)
      //      logger.info(s"DataFilePath: [${dataFilePath}] and ExpectedRecordCount: [${expectedRecordCount}]")
      //      logger.info("DataFilePath: [{}] and ExpectedRecordCount: [{}]", dataFilePath, expectedRecordCount.toString, "")

      def getPureDataFrame(feedPath: String): DataFrame = {

        logger.info("data file path to process: [{}]", feedPath)
        val dataFilePath = feedPath.split(",", -1).map(_.trim)
        val fileFormat = if (StringUtils.isNonEmpty(paramMap(DataFile_Format))) paramMap(DataFile_Format) else ""

        fileFormat.toLowerCase() match {
          case DELIMITED | CSV | TSV => DelimitedParser().parse(dataFilePath: _*)
          case JSON =>
            if (Enums.StorageFormat.withName(paramMap(Constants.Storage_Format)) equals Enums.StorageFormat.JSON_TEXT)
              JsonTextParser.parse(dataFilePath: _*)
            else
              JsonParser.parse(dataFilePath: _*)
          case FIXED_WIDTH => new FixedWidthParser().parse(dataFilePath: _*)
          case _           => throw new IngestionException("Undefined file format found: " + fileFormat)
        }
      }

      /**
       * Either of the below call will work fine
       */
      val df = getPureDataFrame("C:\\Z\\" + map("file"))

      println("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
      df.show(500, false)
      Ingestion.proceed(df)
      println("processing over.!")
    } catch {
      case ex: IngestionException =>
        logger.error("Ingestion Exception Caught: {}", "", ex.getMessage, ex)
      case ex: Exception =>
        logger.error("Exception Caught: {}", "", ex.getMessage, ex)
    }
  }

  def readORCFile(file1: String) {
    val spark = SparkUtil.getSparkSessionForHive("read-orc")

    //val file1 = "./hadoop_log_dir/spark_orc/"
    val csvDF1 = spark.read.format("orc").load(file1)

    logger.info("good df schema:")
    csvDF1.printSchema
    //csvDF1.collect().foreach(println)
    csvDF1.createOrReplaceTempView("main_claim_table")
    spark.sql("select claimant_hire_date, return_to_work_date, * from main_claim_table where myorg_claim_number = 'CN123967858a0005760'").show(100, false)
    //csvDF1.filter(csvDF1("myorg_claim_number") === "CN123967858a0005760").show(100, false)
    val count1 = csvDF1.count
    logger.info("good count:{}", count1)

    //    import scala.collection.JavaConverters._
    //
    //    val files = FileUtils.listFiles(new File(file1), null, false).asScala.filter(f => f.getName.endsWith(".orc"))
    //    //if (files != null && files.size > 0) {
    //    val file = "./hadoop_log_dir/rejected_records_orc"
    //
    //    val csvDF = spark.read.format("orc").load(file)
    //    logger.info("bad df schema:")
    //    csvDF.printSchema
    //    csvDF.show(100, false)
    //
    //    println("\n\n                 INGESTION FAILED \n\n")
    //
    //    val count = csvDF.count
    //    logger.info("bad count:{}", count)
    //}
  }

  def parseCatalog(catalog: String): Tuple3[scala.collection.mutable.Map[String, String], List[FeedDataFileColumn], List[FeedMetaFileColumn]] = {
    val file = new File("C:\\Z\\" + catalog + ".json")
    val fileString = "C:\\Z\\" + catalog + ".json"
    val json = Source.fromFile(fileString).getLines().mkString
    val data = JsonMethods.parse(json)
    val (paramMap, dataFileColList, metaFileColList) = CatalogReader.readCatalogJson(data)
    (paramMap, dataFileColList, metaFileColList)
  }

  def poc() {
    val spark = SparkUtil.getSparkSession("poc", "*")
    val inputRDD = spark.sparkContext.textFile("src/main/resources/input.json")

    val jsonPaths = Seq("$.id", "$.address[*].city", "$.address[*].State")
    val schema = Seq("id", "city", "state")
    //val jsonPaths = Seq("$._id", "$.record_name", "$.type", "$.record_type", "$.status", "$.company_number", "$.description", "$.sov_field_mapping_id", "$.user_id", "$.as_of_date", "$.sov_raw_id", "$.last_updated_timestamp", "$.last_modified_by", "$.version", "$.created_by", "$.test_column.a", "$.test_column.b[*]", "$.test_column.c[1][*]", "$.test_column.c[0]")
    //val schema = Seq("id", "rec_name", "t_type", "rec_type", "status", "comp_num", "desc", "sov_field_mapping_id", "user_id", "as_of_date", "sov_raw_id", "last_updated_timestamp", "last_modified_by", "version", "created_by", "test_col_a", "test_col_b", "test_col_c_arr", "test_column_c_c1")
    val bcSchema = spark.sparkContext.broadcast(jsonPaths)

    import com.jayway.jsonpath.Configuration
    import com.jayway.jsonpath.JsonPath

    val rdd = inputRDD.mapPartitions(iter => {
      val list = iter.toList
      val sch = bcSchema.value
      //val fields = new Array[Any](sch.length)

      val mapped = list.map(record => {
        import com.jayway.jsonpath.Option
        import scala.collection.JavaConversions.setAsJavaSet
        import scala.collection.mutable.MutableList
        val options = scala.collection.mutable.Set[Option](
          Option.AS_PATH_LIST, Option.ALWAYS_RETURN_LIST, Option.SUPPRESS_EXCEPTIONS)

        val conf = Configuration.builder().options(options).build()
        val jsonPaths = JsonPath.using(conf).parse(record)
        val json = JsonPath.parse(record)

        val container = new Array[ListBuffer[Any]](sch.length)
        var maxLen = 0
        sch.zipWithIndex.foreach { pair =>

          val fields = ListBuffer[Any]()
          val pathList: net.minidev.json.JSONArray = jsonPaths.read(pair._1)
          if (!pathList.isEmpty()) {
            val arrays = pathList.toArray().asInstanceOf[Array[Object]]
            arrays.foreach(path => {
              val valOpt = scala.util.Try(json.read[String](path.toString)).toOption
              fields += (if (valOpt.isDefined) valOpt.get else null)
            })
            maxLen = if (fields.size > maxLen) fields.size else maxLen
            container(pair._2) = fields
          }
        }
        println(container)
        container.foreach(list => {
          for (i <- list.size until maxLen) {
            list += list.last
          }
        })
        val transposed = container.toList.transpose
        println(transposed)
        println("greatest column length: " + maxLen)
        transposed.map(list => { val seq = list.toSeq; Row.fromSeq(list.toSeq) })
      }).flatten
      mapped.iterator
    })
    val collect = rdd.collect()
    println("rdd.collect length: " + collect.length)
    collect.foreach(println)
    val df = spark.createDataFrame(rdd, StructType(schema.map(sch => StructField(sch, StringType, true))))
    df.printSchema()
    df.show(false)
  }
}
```
