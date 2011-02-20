---
layout: post
title: Adding Inheritance to AutoMapper
metaTitle: Adding Inheritance to AutoMapper
description: Jake shares a modified version of AutoMapper which inherits mappings when you include Derived types
revised: 2011-02-20
date: 2010-10-31
categories: [automapper,inheritance,open-source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /automapperinheritance/
summary: | 
  

---
In my current project, we are leveraging AutoMapper a lot to map our Domain to Dto's. The major problem we are facing is that our mappings are getting quite complex and bloated, especially for flattened Dtos or Summaries which have properties that cover all the properties from the concrete types they come from.
It seemed unnatural that I could define a mapping that looked like:

    Mapper.CreateMap<ItemBase, ItemSummaryDto>()
        .ForMember(d => d.Description, m => m.MapFrom(s => s.ItemSummary))
        .Include<GeneralItem, GeneralItemSummaryDto>()
        .Include<SpecificItem, SpecificItemSummaryDto>();
    Mapper.CreateMap<GeneralItem, GeneralItemSumaryDto>()
        .ForMember(d => d.GeneralProperty, m => m.MapFrom(s => s.Something));
    Mapper.CreateMap<SpecificItem, SpecificItemSumaryDto>()
        .ForMember(d => d.SpecificProperty, m => m.MapFrom(s => s.Something));

And have the rest of the properties map by convention. The problem is that the .Include function is only used for type resolution. i.e.
    var dto = Mapper.Map<ItemBase, ItemSummaryDto>(new GeneralItem());
    Assert.IsInstanceOfType<GeneralItemSummaryDto>(dto); // Passes

If we validate our mapping we will find that Description is not mapped on GeneralItemSummaryDto and SpecificItemSummaryDto. This means out mapping ends up being:

    Mapper.CreateMap<ItemBase, ItemSummaryDto>()
        .Include<GeneralItem, GeneralItemSummaryDto>()
        .Include<SpecificItem, SpecificItemSummaryDto>();
    Mapper.CreateMap<GeneralItem, GeneralItemSumaryDto>()
        .ForMember(d => d.Description, m => m.MapFrom(s => s.ItemSummary))
        .ForMember(d => d.GeneralProperty, m => m.MapFrom(s => s.Something));
    Mapper.CreateMap<SpecificItem, SpecificItemSumaryDto>()
        .ForMember(d => d.Description, m => m.MapFrom(s => s.ItemSummary))
        .ForMember(d => d.SpecificProperty, m => m.MapFrom(s => s.Something));

We have to duplicate the mapping of the Description property on both the items. For my current project this means our mappings are far more complicated than they need to be, and have massive amounts of duplicated ForMember(d=>d.Blah, m=>m.Ignore()) mappings because we have many flattened summary dtos which have properties from many of the concrete types that are mapped.

Unfortunately it ended up being quite a difficult task, and with our complex mappings my initial implementations resulted in a few issues. I am pretty sure I have got most of them and have created quite a few unit tests covering it. Especially with collections of a base type, holding different concrete types.

Is anyone interested in this functionality? If so let me know and I will create a proper Fork on Github and share my changes. I still need to cleanup the unit tests and the code a little bit as the AutoMapper codebase is really nice and I don't want to make it messy.
