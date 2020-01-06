# Azure Scheduled Task to sync-up Sql Server to End-Point

## Scope

* Dotnet core C#
* Sql Server CDC enabled
* Connect to a sql server table to fetch all the record since the last processed change
* Post a sync message to Azure Event Hub for every record changed since then
* Connect to the end-point to fetch all records in chunks
* For each record from the end-point verify it exists in Sql Server
* For records that are not found in Sql Server post a message to Event Hub to delete the record
* The code can be run from from a command line for initial full sync and from Azure Scheduled Task for the ongoing sync
* Last processed change to be stored in a sql server table
* Unit tests
* Performance test
* Markdown documentaion

### [End-point Spec](https://app.swaggerhub.com/apis/vkhazin/trgos-faretypes/1.0.0)

### Sql Server

* Sql Server Source Table DDL:
```
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [dbo].[FareTypes](
	[FareTypeId] [int] NOT NULL,
	[FareTypeAbbr] [varchar](10) NULL,
	[Description] [varchar](30) NULL,
	[FlatFareAmt] [real] NULL,
	[FlatRate] [real] NULL,
	[DistanceRate] [real] NULL,
	[PolygonRate] [real] NULL,
	[TimeRate] [real] NULL,
	[PolyTypeId] [int] NULL,
	[PrepaidReq] [char](2) NULL,
	[FareAdjustDiffZon] [float] NULL,
	[FareAdjustSameZon] [float] NULL,
	[DistanceLimitSameZon] [float] NULL,
	[DistanceLimitDiffZon] [float] NULL,
	[NumCode] [smallint] NULL,
	[FareMode] [smallint] NULL,
	[FareCalcType] [smallint] NULL,
	[VariationOf] [int] NULL,
	[MinFare] [real] NULL,
	[MaxFare] [real] NULL,
	[RoundFare] [tinyint] NULL,
	[DistanceLimit] [int] NULL,
	[RoundDirection] [smallint] NULL,
	[Accuracy] [real] NULL,
	[InActive] [smallint] NULL,
	[Timestamp] [datetime2] DEFAULT SYSUTCDATETIME ( )
 CONSTRAINT [pkFareTypes] PRIMARY KEY CLUSTERED 
(
	[FareTypeId] ASC
) WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO
```
* Sql Server sample data:
```
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (1, N'STA', N'Standard Fare', NULL, 2, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 1, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (2, N'CON', N'Concessionary Fare', NULL, 1, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 2, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (3, N'CSF', N'Child Standard Fare', NULL, 1, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 3, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (4, N'CCF', N'Child Concessionary Fare', NULL, 0.5, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 4, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (5, N'ESC', N'Escort Fare', NULL, 0.75, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 5, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
INSERT [dbo].[FareTypes] ([FareTypeId], [FareTypeAbbr], [Description], [FlatFareAmt], [FlatRate], [DistanceRate], [PolygonRate], [TimeRate], [PolyTypeId], [PrepaidReq], [FareAdjustDiffZon], [FareAdjustSameZon], [DistanceLimitSameZon], [DistanceLimitDiffZon], [NumCode], [FareMode], [FareCalcType], [VariationOf], [MinFare], [MaxFare], [RoundFare], [DistanceLimit], [RoundDirection], [Accuracy], [InActive]) VALUES (6, N'FOC', N'Free Of Charge', NULL, 0, 0, 0, 0, 0, NULL, 0, 0, 0, 0, 6, 0, 0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL)
GO
```
* Sql Server Tracking Table DDL
```
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [dbo].[CdcTracking](
	[Table] [varchar(255)] NOT NULL,
	[LastProcessedTimestamp] [datetime2] NOT NULL
 CONSTRAINT [pkCdcTracking] PRIMARY KEY CLUSTERED 
(
	[FareTypeId] ASC
) WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO
```

### Event Hub Message Format

```
{
  "table": "FareTypes",
  "action": "delete|sync",
  "timestamp": "2002-10-02T15:00:00.05Z",
  "id": "single field pk field value",
  "data": {
	  "FareTypeId": 1,
	  "FareTypeAbbr": "STA",
	  "Description": "Standard Fare",
	  "FlatRate": 2.0,
	  "DistanceRate": 0.0,
	  "PolygonRate": 0.0,
	  "TimeRate": 0.0,
	  "PolyTypeId": 0,
	  "FareAdjustDiffZon": 0.0,
	  "FareAdjustSameZon": 0.0,
	  "DistanceLimitSameZon": 0.0,
	  "DistanceLimitDiffZon": 0.0,
	  "NumCode": 1,
	  "FareMode": 0,
	  "FareCalcType": 0,
	  "SourceTimestamp": "2002-10-02T15:00:00.05Z"
  }
}
```

