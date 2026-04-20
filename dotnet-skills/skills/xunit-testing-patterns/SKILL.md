---
name: xunit-testing-patterns
description: xUnit testing patterns for .NET - unit tests, integration tests, mocking, test data builders. Use when writing tests for services and controllers.
---

# xUnit Testing Patterns

## When to Use This Skill

Use this skill when:
- Writing unit tests for services
- Testing domain logic
- Mocking dependencies
- Creating integration tests
- Building test data

## Unit Test Structure

### Service Unit Test

```csharp
public class SocialPostServiceTests
{
    private readonly Mock<ISocialPostRepository> _repositoryMock;
    private readonly Mock<ICurrentUserService> _currentUserMock;
    private readonly Mock<IAuditService> _auditMock;
    private readonly SocialPostService _sut;

    public SocialPostServiceTests()
    {
        _repositoryMock = new Mock<ISocialPostRepository>();
        _currentUserMock = new Mock<ICurrentUserService>();
        _auditMock = new Mock<IAuditService>();

        _currentUserMock.Setup(x => x.UserId).Returns(Guid.NewGuid());
        _currentUserMock.Setup(x => x.CompanyId).Returns(Guid.NewGuid());

        _sut = new SocialPostService(
            _repositoryMock.Object,
            _currentUserMock.Object,
            _auditMock.Object,
            Mock.Of<ILogger<SocialPostService>>());
    }

    [Fact]
    public async Task CreateAsync_ValidRequest_ReturnsSocialPostDto()
    {
        // Arrange
        var companyId = Guid.NewGuid();
        var request = new CreateSocialPostRequest
        {
            Type = SocialPostType.Text,
            Content = "Test content"
        };

        _currentUserMock.Setup(x => x.CompanyId).Returns(companyId);

        // Act
        var result = await _sut.CreateAsync(companyId, request, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(SocialPostState.Draft, result.State);
        Assert.Equal(request.Content, result.Content);

        _repositoryMock.Verify(
            x => x.AddAsync(It.IsAny<SocialPost>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task ApproveAsync_UserWithoutSignoffCapability_ThrowsForbiddenException()
    {
        // Arrange
        var companyId = Guid.NewGuid();
        var postId = Guid.NewGuid();

        _currentUserMock.Setup(x => x.HasSignoffCapability).Returns(false);

        // Act & Assert
        await Assert.ThrowsAsync<ForbiddenException>(
            () => _sut.ApproveAsync(companyId, postId, CancellationToken.None));
    }

    [Fact]
    public async Task ApproveAsync_PostNotFound_ReturnsFalse()
    {
        // Arrange
        var companyId = Guid.NewGuid();
        var postId = Guid.NewGuid();

        _currentUserMock.Setup(x => x.HasSignoffCapability).Returns(true);
        _repositoryMock
            .Setup(x => x.GetByIdAsync(postId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((SocialPost?)null);

        // Act
        var result = await _sut.ApproveAsync(companyId, postId, CancellationToken.None);

        // Assert
        Assert.False(result);
    }

    [Fact]
    public async Task ApproveAsync_ValidPost_UpdatesStateAndReturnsTrue()
    {
        // Arrange
        var companyId = Guid.NewGuid();
        var post = SocialPostBuilder.Create()
            .WithCompanyId(companyId)
            .WithState(SocialPostState.AwaitingApproval)
            .Build();

        _currentUserMock.Setup(x => x.HasSignoffCapability).Returns(true);
        _repositoryMock
            .Setup(x => x.GetByIdAsync(post.Id, It.IsAny<CancellationToken>()))
            .ReturnsAsync(post);

        // Act
        var result = await _sut.ApproveAsync(companyId, post.Id, CancellationToken.None);

        // Assert
        Assert.True(result);
        Assert.Equal(SocialPostState.Approved, post.State);

        _repositoryMock.Verify(
            x => x.UpdateAsync(post, It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

## Test Data Builders

### Builder Pattern for Test Data

```csharp
public class SocialPostBuilder
{
    private Guid _id = Guid.NewGuid();
    private Guid _companyId = Guid.NewGuid();
    private Guid _createdById = Guid.NewGuid();
    private SocialPostType _type = SocialPostType.Text;
    private SocialPostState _state = SocialPostState.Draft;
    private string? _content = "Test content";
    private bool _isAiGenerated = false;

    public static SocialPostBuilder Create() => new();

    public SocialPostBuilder WithId(Guid id)
    {
        _id = id;
        return this;
    }

    public SocialPostBuilder WithCompanyId(Guid companyId)
    {
        _companyId = companyId;
        return this;
    }

    public SocialPostBuilder WithState(SocialPostState state)
    {
        _state = state;
        return this;
    }

    public SocialPostBuilder WithType(SocialPostType type)
    {
        _type = type;
        return this;
    }

    public SocialPostBuilder WithContent(string? content)
    {
        _content = content;
        return this;
    }

    public SocialPostBuilder AsAiGenerated()
    {
        _isAiGenerated = true;
        return this;
    }

    public SocialPost Build()
    {
        var post = SocialPost.Create(_companyId, _createdById, _type, _content, null, _isAiGenerated);

        // Use reflection to set state for testing (normally state transitions would be called)
        if (_state != SocialPostState.Draft)
        {
            var stateProperty = typeof(SocialPost).GetProperty("State");
            stateProperty?.SetValue(post, _state);
        }

        return post;
    }
}

public class CompanyBuilder
{
    private string _name = "Test Company";
    private string _clientId = "UK-1234";
    private string? _crmId = null;

    public static CompanyBuilder Create() => new();

    public CompanyBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public CompanyBuilder WithClientId(string clientId)
    {
        _clientId = clientId;
        return this;
    }

    public Company Build() => Company.Create(_name, _clientId, _crmId);
}
```

## Domain Entity Tests

```csharp
public class SocialPostTests
{
    [Fact]
    public void Create_ValidParameters_CreatesDraftPost()
    {
        // Arrange
        var companyId = Guid.NewGuid();
        var userId = Guid.NewGuid();

        // Act
        var post = SocialPost.Create(companyId, userId, SocialPostType.Text, "Content");

        // Assert
        Assert.NotEqual(Guid.Empty, post.Id);
        Assert.Equal(companyId, post.CompanyId);
        Assert.Equal(userId, post.CreatedById);
        Assert.Equal(SocialPostState.Draft, post.State);
        Assert.Equal("Content", post.Content);
    }

    [Fact]
    public void SubmitForApproval_DraftPost_ChangesStateToAwaitingApproval()
    {
        // Arrange
        var post = SocialPostBuilder.Create()
            .WithState(SocialPostState.Draft)
            .Build();

        // Act
        post.SubmitForApproval();

        // Assert
        Assert.Equal(SocialPostState.AwaitingApproval, post.State);
    }

    [Fact]
    public void SubmitForApproval_NonDraftPost_ThrowsInvalidOperationException()
    {
        // Arrange
        var post = SocialPostBuilder.Create()
            .WithState(SocialPostState.Approved)
            .Build();

        // Act & Assert
        Assert.Throws<InvalidOperationException>(() => post.SubmitForApproval());
    }

    [Theory]
    [InlineData(SocialPostState.Draft)]
    [InlineData(SocialPostState.Approved)]
    [InlineData(SocialPostState.Published)]
    public void Approve_NonAwaitingApprovalPost_ThrowsInvalidOperationException(SocialPostState state)
    {
        // Arrange
        var post = SocialPostBuilder.Create()
            .WithState(state)
            .Build();

        // Act & Assert
        Assert.Throws<InvalidOperationException>(() => post.Approve());
    }

    [Fact]
    public void Approve_AwaitingApprovalPost_ChangesStateToApproved()
    {
        // Arrange
        var post = SocialPostBuilder.Create()
            .WithState(SocialPostState.AwaitingApproval)
            .Build();

        // Act
        post.Approve();

        // Assert
        Assert.Equal(SocialPostState.Approved, post.State);
    }
}
```

## Test Organisation

### Test Class Structure

```csharp
// Tests/GPTW.Plus.Application.Tests/Services/SocialPostServiceTests.cs
namespace GPTW.Plus.Application.Tests.Services;

public class SocialPostServiceTests
{
    public class CreateAsync : SocialPostServiceTests
    {
        [Fact]
        public async Task ValidRequest_ReturnsSocialPostDto() { }

        [Fact]
        public async Task InvalidCompany_ThrowsForbiddenException() { }
    }

    public class ApproveAsync : SocialPostServiceTests
    {
        [Fact]
        public async Task UserWithSignoffCapability_ApprovesPost() { }

        [Fact]
        public async Task UserWithoutSignoffCapability_ThrowsForbidden() { }
    }

    public class DeleteAsync : SocialPostServiceTests
    {
        [Fact]
        public async Task ExistingPost_SoftDeletesAndReturnsTrue() { }

        [Fact]
        public async Task NonExistingPost_ReturnsFalse() { }
    }
}
```

## Async Testing

```csharp
[Fact]
public async Task GetAllAsync_ReturnsFilteredPosts()
{
    // Arrange
    var companyId = Guid.NewGuid();
    var posts = new List<SocialPost>
    {
        SocialPostBuilder.Create().WithCompanyId(companyId).WithState(SocialPostState.Draft).Build(),
        SocialPostBuilder.Create().WithCompanyId(companyId).WithState(SocialPostState.Published).Build()
    };

    _repositoryMock
        .Setup(x => x.GetByCompanyAsync(companyId, SocialPostState.Draft, It.IsAny<CancellationToken>()))
        .ReturnsAsync(posts.Where(p => p.State == SocialPostState.Draft).ToList());

    var filter = new SocialPostFilter { State = SocialPostState.Draft };

    // Act
    var result = await _sut.GetAllAsync(companyId, filter, CancellationToken.None);

    // Assert
    Assert.Single(result);
    Assert.All(result, p => Assert.Equal(SocialPostState.Draft, p.State));
}
```

## References

- [xUnit Documentation](https://xunit.net/docs/getting-started/netcore/cmdline)
- [Moq Documentation](https://github.com/moq/moq4)
- [Unit Testing Best Practices](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
