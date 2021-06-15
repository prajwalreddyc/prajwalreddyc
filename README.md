// Databricks notebook source
//This is a band aid until we upgrade to a new Scala version on the cluster - remove once upgrade is completed.
spark.conf.set("spark.sql.crossJoin.enabled", "true")

// COMMAND ----------

// MAGIC %run /MDP/050_BaseToAtomic/010_GenerateConstants_BaseToAtomic

// COMMAND ----------

val stageInProcess = "BaseToAtomic"
val sourceTable = "CMC_CSPI_CS_PLAN"
val targetTable = "Hp_Bnft_Set"
val strDomainName = "Health Plan Benefit Set Type Code"
val strCommonCode = "Health Plan Benefit Set Type Code"
val facetsBasePointPath = baseMountPointPath + "Facets/"
val sourcePath = facetsBasePointPath + sourceTable + "/"
val targetPath = atomicMountPointPath + targetTable

// COMMAND ----------

val schema = StructType(
StructField("Hp_Bnft_Set_Sk",LongType, false) ::
StructField("Hp_Bnft_Set_Bk",StringType, false) ::
StructField("Tenant_Sk",LongType, false) ::
StructField("Load_Info_Sk",LongType, false) ::
StructField("Type_Cd_Sk",LongType, false) ::   Nil)

 val dfDeltaRead = if(DeltaTable.isDeltaTable(spark, targetPath))
                                           {spark.read.format("delta").load(targetPath)}
                                     else{spark.createDataFrame(spark.sparkContext.emptyRDD[Row], schema)}

if(!(DeltaTable.isDeltaTable(spark, targetPath))){dfDeltaRead.write.format("delta").mode("append").save(targetPath)}

// COMMAND ----------

//Apply any Where filters on the anchor model logic here at the read to avoid reading in extra data
val dfBaseRead = spark.read.format("delta").load(sourcePath)

// COMMAND ----------

val dfIdentifyNewRecordsJoined = getLoadInfoAuditDelta(dfBaseRead,
                                                  "file",
                                                   Map(
                                                     "StageInProcess"    -> stageInProcess ,
                                                     "ParentSubjectArea" -> "HealthPlan",
                                                     "SubjectArea"       -> "BenefitSet",
                                                     "SourceFileName"    -> $"FileName",
                                                     "TargetFileName"    -> targetTable
                                                   ))

// COMMAND ----------

val dfLoadInfoSkSource = dfIdentifyNewRecordsJoined.as("dfIdentifyNewRecordsJoined")
                                  .groupBy($"dfIdentifyNewRecordsJoined.FileName",$"dfIdentifyNewRecordsJoined.FileDate")
                                  .agg(count($"dfIdentifyNewRecordsJoined.*").cast("Integer").as("SourceRecordCount"),
                                       min($"dfIdentifyNewRecordsJoined.dTransaction").as("DataStartDateTime"),
                                       max($"dfIdentifyNewRecordsJoined.dTransaction").as("DataEndDateTime"))

// COMMAND ----------

val dfLoadInfoSkReturned = generateSk(dfLoadInfoSkSource,loadInfoAudtPath,"Load_Info_Sk")

// COMMAND ----------

val anchorTablePass = dfIdentifyNewRecordsJoined.as("dfIdentifyNewRecordsJoined")
                                                .join(dfFinalSks.as("dfFinalSks"), lit("Tenant|CareSource") === $"dfFinalSks.Tenant_Bk","inner")
                                                .join(dfLoadInfoSkReturned.as("dfLoadInfoSkReturned"), 
                                                        Seq("FileName","FileDate"),"inner")
                                                .withColumn("Dmn_Nm",lit(strDomainName))
                                                .withColumn("Cmn_Cd",lit(strCommonCode))
                                                .withColumn("Descr",col("Cmn_Cd"))
                                                .withColumn("SourceColumnName",col("Descr"))
                                                .select($"Src_Cd_Sk", //Begin System Columns 
                                                        $"Tenant_Sk", 
                                                        $"dfLoadInfoSkReturned.Load_Info_Sk",
                                                        $"StageInProcess",
                                                        $"ParentSubjectArea",
                                                        $"SubjectArea",
                                                        concat(lit(sourcePath),$"FileName").as("SourcePath"),
                                                        $"FileName".as("SourceFileName"),
                                                        lit(targetPath).as("TargetPath"),
                                                        $"TargetFileName", 
                                                        $"SourceRecordCount",
                                                        $"DataStartDateTime",
                                                        $"DataEndDateTime",
                                                        $"dfIdentifyNewRecordsJoined.FileName",
                                                        $"dfIdentifyNewRecordsJoined.FileDate",//End System Columns
                                                        $"SourceColumnName", //Begin Common Code Columns
                                                        $"Dmn_Nm", 
                                                        $"Descr", 
                                                        $"Cmn_Cd", //End Common Code Columns & Cmn_Cd is Part of Business Key for Anchor
                                                        $"GRGR_CK",
                                                        $"CSCS_ID",
                                                        $"CSPD_CAT",
                                                        $"CSPI_ID",
                                                        $"CSPI_EFF_DT")
                                                .distinct()

// COMMAND ----------

val rollbackVersion = getDeltaTableCurrentVersion(spark,targetPath)
try
         {
           
           //Write to the target table ; next read from target table and prepare data for etlmetadata table 
            getAtomicAnchorSk(anchorTablePass,targetTable)
             
           //createLoadInfoAudt - runs above function to generate the Load_Info_Audt insert dataframe
           //writeLoadInfoAudt - actually writes to loadInfoAudtPath which is String = /mnt/integration/atomic/Load_Info_Audt
             writeLoadInfoAudt(createLoadInfoAudt(anchorTablePass,targetPath))
         }
       catch
         {
            
            case e: Exception => 
                    {
                      // Rollback the cleansed Layer Delta Table
                      
                      rollBackDeltaTable(spark, targetPath, rollbackVersion)
                      
                      throw new Exception("Error while writing to the Base Layer, Rolled back " + targetPath, e)
                    }
          } 
