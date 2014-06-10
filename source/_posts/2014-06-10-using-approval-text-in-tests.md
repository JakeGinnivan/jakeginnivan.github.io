## Using Approval Text in tests

I am currently using [TestStack.BDDfy](https://github.com/TestStack/TestStack.BDDfy) as my BDD testing framework. v4 has some great changes which make BDDfy really really awesome.

One of the things I am testing is generated confirmation text based on a number of selected form inputs. I want the approved generated text to be put into the BDDfy output. First steps are to read the approved file, here is a quick snipped which does this:

    protected string GetApproved()
    {
        var namer = new UnitTestFrameworkNamer();
        var basename = string.Format("{0}\\{1}", namer.SourcePath, namer.Name);
        var approvalFilename = new ApprovalTextWriter(string.Empty).GetApprovalFilename(basename);
        var approved = !File.Exists(approvalFilename) ? string.Empty : File.ReadAllText(approvalFilename);
        return approved;
    }

Then I can just add my step in BDDfy `.Then(_ => ApprovedGeneratedConfirmationShouldMatch(sut.DealSummary.ToString()), string.Format("Approved generated confirmation should be:\r\n{0}", GetApproved()))`
    
 